# 07. Three Invoices, and What Actually Distinguishes Them

## The naming is not the confusing part, the actual distinction is

`src/components/invoice` has three sub-modules, `purchase-inovice` (the folder name really is misspelled), `sales-invoice`, and `refund-invoice`. They sound like they might just be three redundant records of the same event, but they genuinely track three different sides of one transaction, and the clearest way to see that is to look at who calls each one, and when.

Grepping for `generateInvoice(` and its interfaces shows both `SalesInvoiceServiceInterface` and `PurchaseInvoiceServiceInterface` get injected into, and called from, the exact same places, the Stripe, CoinGate, Cryptomus, and Unstoppable Domains webhook handlers, plus the Freename registration service, and always at the same moment, right after a payment is confirmed and the domain registration itself succeeds. In `stripe-webhook.service.ts`, the sales invoice is generated first, then, two lines later:

```ts
invoice = await this.salesInvoiceServiceInterface.generateInvoice(mutateDataForSalseInvoce, user.id);
// ...domain registration on chain succeeds...
const purchaseInvoiceData = await mutateOrderToPurchaseInvoiceDto(latestDomainOrderWithDetails);
await this.purchaseInvoiceServiceInterface.generateInvoice(purchaseInvoiceData);
```

## Sales invoice, what the customer paid Endless Domains

`SaleInvoiceEntity` is the customer facing receipt, `actualAmount` (the sticker price before any discount), `paidAmount` (what actually got charged after the coupon, if any), `couponCode`, `paymentType` (`stripe`, `cryptomus`, `coingate`), `domainOwner`, `transactionHash`, and a `domainName` column that is actually an array of `{ domainName, chainName }` objects, one entry per domain in the order. This is the only one of the three that produces an actual downloadable document, not just a database row.

`SalesInvoiceService.generateInvoice` renders a real PDF using `puppeteer` against a handlebars template (`hbs/invoice-template.hbs`, styled with the company's own purple and gold border, logo, and terms link), uploads the rendered PDF to S3, and hands back a `downloadUrl` that is not a direct S3 link, it is a signed backend URL carrying a short lived JWT:

```ts
const token = this.jwtService.sign({ invoiceId: newInvoice.invoiceNumber, purpose: 'invoice-download' }, { secret: this.JWT_SECRET, expiresIn: '1d' });
let downloadUrl = `${this.BACKEND_URL}/sales-invoice/invoice/download?token=${token}`
```

`getInvoiceDownloadByToken` on the receiving end verifies that token, checks its `purpose` claim is specifically `'invoice-download'` (so a token minted for some other purpose in this same JWT secret space could never be repurposed to fetch an invoice), and only then looks up the invoice strictly by the `invoiceId` baked into the token itself, never by anything the caller could pass directly, before streaming the PDF bytes back from S3. The download endpoint itself sits behind `IpTokenRateLimitGuard` rather than a login check, a deliberate choice, because the whole point of a signed, short lived download link is that it should work for someone clicking it from an email, without necessarily being logged back into the site first.

Invoice numbering is the one place in this entire cluster with a real, correctly built database transaction:

```ts
async createInvoice(dto: SaleInvoiceDto): Promise<SaleInvoiceEntity> {
    const maxAttempts = 3;
    let attempt = 0;
    while (true) {
        try {
            return await this.saleInvoiceRepo.manager.transaction(async (manager) => {
                const invoiceNumber = await this.generateInvoiceNumber(manager); // SELECT ... FOR UPDATE
                const repo = manager.getRepository(SaleInvoiceEntity);
                const newInvoice = repo.create({ ...dto, invoiceNumber });
                return repo.save(newInvoice);
            });
        } catch (error: any) {
            attempt += 1;
            if (error?.code === '23505' && attempt < maxAttempts) continue;
            throw error;
        }
    }
}
```

`generateInvoiceNumber` finds the highest existing `S<number>` invoice number and adds one, using `.setLock('pessimistic_write')`, a real row lock, so two concurrent invoice creations cannot both read the same "last number" and generate the same next one. Wrapping the read and the insert in one `manager.transaction` is what makes that lock actually mean something, and the retry loop around a Postgres unique violation (`23505`) is a sensible belt and suspenders on top of it. This is, deliberately, the single clearest positive example in the whole cluster of the discipline described conceptually in the LearnBridge orders and payments planning notes, a sequence collision here would visibly break something (two invoices with the same number), so it got the proper transaction it needed, while most of the sequential, un-transacted writes elsewhere in this cluster get away with it because their individual steps are more forgiving of a partial failure.

## Purchase invoice, what Endless Domains itself paid for the domain

`PurchaseInvoiceEntity` looks superficially similar, `provider`, `domainAmount`, `actualAmount`, `discountPrcentage`, `chainMinted`, `domainName` (a plain string array here, not the richer objects the sales invoice uses), but it is not customer facing at all, there is no PDF, no S3 upload, no download link, just a plain database row with an auto generated `P<number>` invoice id. Reading `mutateOrderToPurchaseInvoiceDto` in `PurchaseInvoiceService`, the number it actually computes is telling:

```ts
if (provider === 'UD') {
    amount = orderData.domainDetailList?.reduce((sum, domain) => {
        const tld = domain.domainName.split('.').pop();
        let finalPrice = domain.price;
        if (tld === 'og') {
            finalPrice = domain.price * 0.5;
        } else {
            const tldPercentage = UDTLDLIST[tld] || 0;
            const discount = (tldPercentage / 100) * domain.price;
            finalPrice = domain.price - discount;
        }
        return sum + finalPrice;
    }, 0) || 0;
}
```

`UDTLDLIST` is a per TLD commission or wholesale discount table, the price stored here is not what the customer paid, it is what this system computes Endless Domains' own cost basis to be for actually acquiring that domain from the upstream registry, after whatever wholesale arrangement applies to that particular TLD. Put simply, the sales invoice is the revenue side of this transaction, and the purchase invoice is the cost side, generated in the same webhook handler, moments apart, as two halves of the same event recorded from two different accounting perspectives. That distinction is not stated explicitly anywhere in the code's comments, it has to be inferred from what each one actually computes and who reads it, but it fits every piece of evidence, the naming, the near identical calling sites, and the very different numbers each one produces for the exact same order.

## Refund invoice, the paper trail for money going back out

`RefundInvoiceEntity` is the smallest and simplest of the three, `orderId` (unique), `domainName[]`, `walletAddress`, `paymentMode`, `txnId`, `amount`, `currency`, created through a plain, un-transacted create and save with no PDF generation at all, purely a database record of a refund that happened. It is worth not confusing this with the `refund` module covered in `08`, that module tracks the refund's own lifecycle (pending, processing, rejected, completed), this one is just the resulting receipt once a refund is actually issued, the refund equivalent of the sales invoice.

## Frontend note

A frontend "download my invoice" button is, on the backend, a request for a signed, short lived, purpose scoped token that gets exchanged for a PDF fetched fresh from S3 on every click, not a static file the frontend ever holds a direct link to, which is exactly the pattern worth reaching for any time a document needs to be shareable (an email link, a support ticket) without requiring the recipient to already be logged in. And the purchase invoice quietly sitting alongside every sales invoice is a good reminder that a real commerce backend is usually keeping two ledgers at once, what the customer paid, and what the business itself paid to be able to sell that thing at all, even when only one of those two ever reaches a UI.
