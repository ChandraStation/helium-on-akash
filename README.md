# Helium on Akash

This repository includes everything needed to run a Helium validator on Akash. The container will connect to an S3 bucket to upload/download the swarm_key on boot. 

The main files to understand are:

- `Dockerfile` - Installs AWS CLI on top of the [Helium validator docker image](https://quay.io/team-helium/validator) and sets boot.sh to run whenever the container starts
- `boot.sh` - Downloads the swarm_key from S3 (if it exists), starts the miner and prints the address. It then uploads the swarm_key if it didn't download it earlier (new miner).
- `deploy.yml` - Example/working Akash deployment configuration. This is setup to use my image which may or may not be up to date. See below to create and host your own image if needed

## Requirements

- [S3 bucket and IAM user](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-walkthroughs-managing-access-example1.html#grant-permissions-to-user-in-your-account-step1), with access key and secret
- [Dockerhub account](https://hub.docker.com/signup) to host your own container image, if required
- [Testnet wallet](https://docs.helium.com/mine-hnt/validators/validator-deployment-guide#create-testnet-wallet) to stake to your validator and claim it

## Run the container locally

```
docker run --publish 2154:2154/tcp -e AWS_ACCESS_KEY=mykey -e AWS_SECRET_KEY=mysecret -e S3_KEY_PATH=mybucket/miner1/swarm_key tombeynon/helium-on-akash
```

## Deployment

You can deploy the validator on Akash using the example deploy.yml. Note that to use your own image which you can keep up to date, check the next section. 

Either clone this repository or create a `deploy.yml` file. Enter your S3 bucket and IAM credentials into the `env` section. If you have a swarm_key already, make sure this is uploaded to S3 in the same location as S3_KEY_PATH.

Deploy as [per the docs](https://docs.akash.network/guides/deploy) or using a [deploy UI](https://github.com/tombeynon/akash-deploy).

Once the container is deployed, check the logs to see your address once the server starts (can take a while). If your swarm_key didn't exist in S3 before, the new one should have been uploaded. Subsequent deploys using the same S3 details will now use the same swarm_key.

## Create a Helium Testnet Wallet 

Install the [Helium CLI Wallet](https://github.com/helium/helium-wallet-rs).

Once you have your Helium CLI Wallet installed locally, it's time to create your Testnet Wallet. Run the following command to create it. (This command format assumes you're using the executable. If you've built the wallet from source it'll look slightly different.)

`$ helium-wallet create basic --network testnet`

You'll be prompted to supply a new passphrase to complete it. This is used to encrypt/decrypt the wallet.key file, and is needed to sign transactions. Don't lose it.

This command will produce a wallet.key file on your machine, along with output similar to the following:
```
+-----------------------------------------------------+---------+--------+------------+
| Address                                             | Sharded | Verify | PwHash     |
+-----------------------------------------------------+---------+--------+------------+
| 1aP7nm6mGLMFtgiHQQxbPgKcBwnuQ6ehgTgTN8zjFuxByzJ8eA5 | false   | true   | Argon2id13 |
+-----------------------------------------------------+---------+--------+------------+
```
Next, run the info command to get all the relevant details.

`$ helium-wallet info`

The output will look similar to this:
```
+--------------------+-----------------------------------------------------+
| Key                | Value                                               |
+--------------------+-----------------------------------------------------+
| Address            | 1aP7nm6mGLMFtgiHQQxbPgKcBwnuQ6ehgTgTN8zjFuxByzJ8eA5 |
+--------------------+-----------------------------------------------------+
| Network            | testnet                                             |
+--------------------+-----------------------------------------------------+
| Type               | ed25519                                             |
+--------------------+-----------------------------------------------------+
| Sharded            | false                                               |
+--------------------+-----------------------------------------------------+
| PwHash             | Argon2id13                                          |
+--------------------+-----------------------------------------------------+
| Balance            | 0.00000000                                          |
+--------------------+-----------------------------------------------------+
| DC Balance         | 0                                                   |
+--------------------+-----------------------------------------------------+
| Securities Balance | 0.00000000                                          |
+--------------------+-----------------------------------------------------+
```
Note the Balance | 0.00000000.

The next step is to acquire some Testnet Tokens (TNT). 

## Acquire TNT from the Testnet Faucet

Running a Validator requires a stake. This stake is 10000 tokens per Validator. For the Testnet we are using TNTs.

To acquire them, head to faucet.helium.wtf and input your the public key from the wallet you just create. Use your public wallet address. If you copy and paste the one above the TNT will be sent to someone else.

Once you've input your address, the Faucet will deliver just over 10000 TNT to your Testnet Wallet. This can take up to 10 minutes so please be patient. Check your wallet balance using the balance command:

`$ helium-wallet balance`

If all went to plan, you'll see this:
```
+-----------------------------------------------------+----------------+--------------+-----------------+
| Address                                             | Balance        | Data Credits | Security Tokens |
+-----------------------------------------------------+----------------+--------------+-----------------+
| 1aP7nm6mGLMFtgiHQQxbPgKcBwnuQ6ehgTgTN8zjFuxByzJ8eA5 | 10005.00000000 | 0            | 0.00000000
```
## Stake your TNT to the Testnet Validator running on Akash
(Make sure to replace the address with your own.)

`helium-wallet validators stake <YOUR VALIDATOR NODES ADDRESS YOU ACQUIRED FROM THE DEPLOYMENT LOGS> 10000 --commit`

After running this, you'll need to input your wallet passphrase to sign the transaction.

And with that, you're done. Congratulations! You're running a Helium Network Validator on the Akash Network.

## Build your own image

There are a couple of reasons to run your own image:

- Akash requires a version specific tag to update the container image. If you use e.g. `latest`, updating the deployment won't pull the latest version of the tag. Helium currently publishes their docker images using the `latest` format only, so to tag the image as required we need to publish our own.
- Testnet moves quickly and I might not keep my image up to date

Create a dockerhub account first, then build the image as follows:

```
git clone git@github.com:tombeynon/helium-on-akash.git
cd helium-on-akash
docker build . -t mydockerhubuser/helium-on-akash:v0.0.1
docker push mydockerhubuser/helium-on-akash:v0.0.1
```

You can then change the `image` value in deploy.yml to your repository and version above.

To update the miner on Akash, run the above to build it with the latest Helium image, incrementing the version number. Then close and re-deploy on Akash using the new version number. This process could be scripted relatively easily.

## Caveats

- Updating the container isn't ideal, this could be improved in the future
- Currently only the swarm_key is synced to S3, meaning the entire blockchain needs to be downloaded each time you run the miner. It takes about 30 minutes currently with the suggested deploy.yml
- There is a delay between 'Starting miner...' and the logs for the miner showing. The miner is running during this time, just the logs don't show. They appear after 5-10 minutes.
- The miner currently bounces from relay to online on the Helium explorer, this might be possible to improve? Though based off our findings it does not impact the nodes opportunities to be elected for consensus.

## References

- https://docs.helium.com/mine-hnt/validators/validator-deployment-guide
- https://explorer.helium.wtf/validators
- https://testnet-api.helium.wtf/v1/validators/{{ADDRESS}}

## Shoutout
shout out to Tom! He played a big roll in making this possible
https://twitter.com/Tom_Beynon
https://github.com/tombeynon
