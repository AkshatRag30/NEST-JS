# 04. Infrastructure plumbing: health checks, provider management, AWS, and IPFS

## The health check, as simple as it looks

`src/components/healthCheck/healthCheck.controller.ts` is genuinely this small.

```ts
@Controller('health')
export class HealthCheckController {
    @Get()
    public async healthCheck(): Promise<Response> {
        return new Response('OK');
    }
}
```

There is no database call, no dependency check, nothing beyond returning a 200 with the message "OK." This is exactly what it looks like, an endpoint an infrastructure component, an AWS Application Load Balancer's target group health check, or an external uptime monitor, pings on a fixed interval to decide whether this particular running instance of the app is alive and should keep receiving traffic, or whether it should be considered unhealthy and cycled out. The module imports `HttpModule` but nothing in the controller actually uses it, a small, harmless leftover rather than a real dependency. If this endpoint ever stops responding, the load balancer would eventually stop routing traffic to that instance, which is precisely the point, it is a canary, not a feature.

## Provider Management: the admin panel behind the "many blockchains" story

Note 01 of the whole notes set already flagged that Endless Domains supports many separate blockchain integrations at once, `ens-integration`, `ud-integration`, `bnb-integration`, and so on. `ProviderManagementModule` (`src/components/provider-management/`) is the admin side control panel over the reference data those integrations all read from, which domain providers and TLDs exist, and which smart contract address backs each one on chain.

`ProviderManagementController` sits behind `SuperAdminAccessGuard` on every route and exposes a small, focused set of admin operations: list every provider and TLD pair with pagination, get aggregated stats (total distinct providers, total distinct TLDs), view the most recently added TLD, look up one provider's full smart contract detail, update a provider's smart contract "registrar" address, and list every TLD in the system flatly. `ProviderManagementUserRepo`, the repository behind all of this, is a clean example of the base repository pattern from note 04 of the wider notes set actually being used as intended.

```ts
export class ProviderManagementUserRepo extends BaseAbstractRepository<DomainProviderAndTldsEntity> implements ProviderManagementRepoInterface {
    constructor(
        @InjectRepository(DomainProviderAndTldsEntity) private readonly providerBaseManagementRepo: Repository<DomainProviderAndTldsEntity>,
        @InjectRepository(DomainProviderSmartContractsEntity) private readonly smartContractRepo: Repository<DomainProviderSmartContractsEntity>
    ) {
        super(providerBaseManagementRepo);
    }
```

It extends `BaseAbstractRepository<DomainProviderAndTldsEntity>` for the five generic operations that class offers, while adding its own more specific query methods (`getOverAllStats`, `getDetailOfOneProvider`, `updateSmartContractAddress`) directly on top, which is exactly the "both styles living side by side in one class" situation note 04 predicted, this module both extends the shared base and writes its own custom queries where the generic five methods are not enough.

This module does not itself talk to any blockchain or registrar. It manages the metadata rows (`DomainProviderAndTldsEntity`, `DomainProviderSmartContractsEntity`, both entities that live inside the `domain` and `moralis` feature areas other agents' notes cover) that every one of the many separate chain integration modules ultimately reads to know which TLDs exist and which contract address to call. If you ever needed to onboard a new TLD under an already supported provider, or point the app at a freshly redeployed smart contract, this admin panel, not a code change, is where that update would actually happen.

## Two small AWS wrappers, and how they differ from `SecretsService`

`src/components/aws/s3/s3.service.ts` and `src/components/aws/sns/sns.service.ts` are both small, focused wrappers around one AWS SDK client each, `S3Client` and `SNSClient`. Neither one has anything to do with `SecretsService` from note 03 despite living in the same general "AWS integration" space, and the distinction is worth being precise about, since all three sit under `@aws-sdk/*` packages and it would be easy to lump them together.

`SecretsService` talks to AWS Secrets Manager to fetch configuration values, API keys, database credentials, the actual contents of the app's environment. `S3Service` and `SnsService` talk to two entirely different AWS products, S3 (file storage) and SNS (a publish and subscribe notification service), to do real, ongoing product work, not configuration bootstrapping.

`S3Service.uploadFile` is the one method almost every other file uploading feature in this app calls through the shared `S3ServiceInterface` token, whether that is a downloadable AI report (note 03), an Excel export from a cron job (note 05), or a user's profile picture handled by a completely different cluster. It takes an `Express.Multer.File`, a target directory prefix, and a bucket name, and returns the S3 object key it wrote to.

```ts
async uploadFile(file: Express.Multer.File, dir: string, bucketName: string, acl?: string): Promise<string> {
    const key = dir + file.originalname;
    const s3 = new S3Client({ region: 'us-east-1' });
    await s3.send(new PutObjectCommand({ Key: key, Bucket: bucketName, Body: file.buffer, ...(acl ? { ACL: acl as ObjectCannedACL } : {}) }));
    return key;
}
```

Note that a brand new `S3Client` is constructed on every single call rather than being built once in the constructor and reused, the same "construct it fresh every time" pattern flagged in the README for this cluster as a small, harmless inefficiency worth noticing rather than a bug. The class also offers `getPresignedUrl` (a temporary, signed download link that expires after 900 seconds by default, useful for letting a browser download a private file directly from S3 without the backend having to proxy the bytes through itself), `getFiles`, `emptyDirectory`, `listHighlightImages`, and `getFileBuffer`.

`SnsService.sendSNS` is much smaller, it publishes a plain text message to one hardcoded topic ARN (`TOPIC_ARN`, read from `ConfigService`), and swallows any error into a log line rather than throwing. SNS in AWS terms is a fan out notification service, one publish can trigger any number of independent subscribers (an email, an SMS, another Lambda function, a queue), so this class is the single point in this codebase where the app hands a message off to that fan out system without needing to know or care who ends up receiving it.

`src/components/aws-secrete/loadSecrets.ts` is a fourth, smaller variation on the exact same "fetch a secret bundle from AWS Secrets Manager" idea already covered in note 03's `SecretsService` and note 04's `TypeOrmConfigService.ensureValues`.

```ts
export default async function loadSecrets(): Promise<Record<string, string>> {
    const secretsManager = new SecretsManager({ region: 'us-east-1' });
    try {
        const data = await secretsManager.getSecretValue({ SecretId: process.env.AWS_MANAGER });
        if (data.SecretString) {
            const secret = JSON.parse(data.SecretString);
            Object.keys(secret).forEach(key => { process.env[key] = secret[key]; });
        }
    } catch (err) {
        // errors intentionally swallowed — app will fail at validation if secrets are missing
    }
    return {};
};
```

Rather than returning the secret bundle as an object the way `SecretsService.getSecret` does, this version mutates `process.env` directly, copying every key from the fetched secret straight onto the running process's own environment variables. A search across the whole codebase for where this function is actually imported turns up nothing beyond its own definition, meaning this is a fourth independent implementation of the same idea that, as far as the code shows, is not currently wired into the app's actual startup path at all. This is exactly the kind of thing note 03 already predicted you would find once you got comfortable enough with a real codebase to go looking, a reasonable looking utility function that turns out to be unused, worth flagging rather than assuming it must be doing something just because it exists.

## IPFS, explained from first principles

IPFS stands for the InterPlanetary File System. If you have only ever worked with a traditional file storage service like S3, the mental model shift is this: instead of asking a specific server "give me the file at this URL," you ask the network "give me the file whose content hashes to this exact fingerprint." The address of a file on IPFS, something like `QmHash123...`, is derived mathematically from the file's own contents, not from which server happens to be hosting it. Any computer on the IPFS network that happens to have a copy of that exact content can serve it back to you, and if the content ever changed even slightly, its address would change too. This is why IPFS is popular in Web3 specifically, an NFT's metadata (its name, image, and attributes) can be pinned to IPFS once, and the NFT's on chain record can simply point at that permanent, content addressed hash forever, rather than pointing at a URL on some company's server that could go offline or get taken down.

"Pinning" is the piece that makes this practically useful. IPFS on its own does not guarantee any particular file stays available forever, a file only stays reachable as long as some computer on the network is still storing and serving it. Pinning services like Pinata exist specifically to be that reliable, always on computer, you upload a file to Pinata, Pinata pins it (keeps a permanent copy and makes it available on the public IPFS network), and hands you back the content hash.

## This codebase's actual IPFS usage: `PinataService`, and the oddly named `ipfs-management`

`src/components/ipfs-server/pinata/pinata.service.ts` is the real, literal IPFS integration in this codebase, a thin wrapper around Pinata's HTTP API using the shared `httpClient` from note 06.

```ts
async pinToIPFS(formData: FormData, metaDataKey: string, metaData: string): Promise<string> {
    formData.append(metaDataKey, metaData);
    const resFile = await httpClient({ method: 'post', url: this.IPFS_URL + '/pinning/pinFileToIPFS', data: formData, headers: { 'Content-Type': 'multipart/form-data', Authorization: this.JWT } });
    return resFile?.data?.IpfsHash;
}
```

`pinToIPFS` uploads a file to Pinata and returns the resulting IPFS hash, `getIPFSHash` looks an existing pin back up by a metadata name, and `unpinFromIPFS` removes one. A search across the codebase shows `PinataModule` is actually consumed by two other feature modules outside this cluster's assigned scope, `nft-collection` (for storing an NFT's metadata and image on IPFS the way the "first principles" section above describes) and `web3/ipfs` (for storing the assets behind a user's parked domain landing page). Both of those live in other clusters, but understanding what `PinataService` itself does is what matters here.

`src/components/ipfs-management/` (`IpfsUserModule`, `IpfsUserService`, `IpfsUserRepo`) is a different, more confusingly named module. Its `fetchAllIpfsData` and `fetchSingleIpfsData` methods (behind `SuperAdminAccessGuard`) are a straightforward admin read over `IpfsEntity` rows, built the same `BaseAbstractRepository` way as `ProviderManagementUserRepo` above. But its other two methods, `createGigapubProject` and `getGigapubStatistic`, have nothing to do with IPFS at all. They call out to a completely separate third party HTTP API, Gigapub, whose URL and bearer token come from `GIGAPUB_API_URL` and `GIGAPUB_TOKEN`.

```ts
async createGigapubProject(data: CreateGigapubProjectDto): Promise<any> {
    const response = await httpClient.post(`${this.GIGAPUB_API_URL}project/create`, data, { headers: { Authorization: `Bearer ${this.GIGAPUB_TOKEN}`, 'Content-Type': 'application/json', 'x-partner-id': '3883' } });
    return response.data;
}
```

`CreateGigapubProjectDto` takes a `telegramUrl`, a `webAppUrl`, and a `name`, which strongly suggests Gigapub is a Telegram mini app publishing and analytics platform, unrelated to decentralized file storage, that happens to have landed inside a folder named for IPFS. The most likely explanation, and a genuinely useful lesson in reading a real codebase, is that this feature was added to a module that already existed for a loosely related reason (parked domain landing pages, which do involve IPFS elsewhere in the app) rather than getting its own home, and the folder name simply never got updated to reflect what actually ended up living inside it. This is exactly the kind of mismatch between a folder's name and its real contents that note 01 of the wider notes set warned you to check for empirically rather than trust blindly, and it is worth remembering the next time a folder name in this codebase seems to promise something the code inside does not quite deliver.
