# Ritual Node Setup Guide

## Deploy a node

### Get a server

To fulfill recommended specs, rent the smallest bare metal server [on Latitude](https://www.latitude.sh/r/5F5E543A), "c2.small.x86," with Ubunutu installed.  

Set up ssh keys so you can access the server via mac Terminal app using [these instructions](https://docs.latitude.sh/docs/ssh).

### Set up an RPC

[Create an account with Alchemy](https://alchemy.com/?r=c57c652206e3703c) using your Gmail or similar.

Create a 'new app' on Base chain.

Save the full URL of the https API key somewhere to use later.

### Set up a fresh wallet

Create a new wallet with a new seed phrase. You will need the private key later, so save that somewhere safe.

Send this wallet 0.03 ETH on Base or more. This is enough to pay for gas and have plenty of ETH left over.

Register the wallet with Ritual by going to https://basescan.org/address/0x8d871ef2826ac9001fb2e33fdd6379b6aabf449c#writeContract

Find the "registerNode" function, enter the address you just created, send tx

Wait 1 hour for a mandatory cooldown period, then find the 'activateNode' function, send tx

### Install the node

This is a modified version of [these instructions](https://docs.ritual.net/infernet/node/deployment) to run a single node locally. Except in our case, 'locally' means on Latitude. 

Using your Terminal, log onto your server. The first command will be something like this, where the number is your server's IP address:

```
ssh ubuntu@123.456.789
```

Immmediately, every time you log in to this server, enter this: `sudo -s`

First, install Docker according to [these instructions from Latitude](https://docs.latitude.sh/docs/docker-for-beginners).

Then clone [this node directory](https://github.com/ritual-net/infernet-deploy) to your server:

```
git clone https://github.com/ritual-net/infernet-deploy
```

Then you need to create a configuration file, which uses your private key and API from earlier. 

```
cd /infernet-deploy/deploy
nano config.json
```
Here's the config file template, modified for simplicity. You need to edit the `rpc_url` and `private_key` fields to match your own from the things you saved earlier. Leave the rest.

```
{
  "log_path": "infernet_node.log",
  "server": {
    "port": 4000
  },
  "chain": {
    "enabled": true,
    "trail_head_blocks": 4,
    "rpc_url": "https://base-mainnet.g.alchemy.com/v2/reeeeeeeeeeeeeeee",
    "coordinator_address": "0x8D871Ef2826ac9001fB2e33fDD6379b6aaBF449c",
    "wallet": {
      "max_gas_limit": 4000000,
      "private_key": "aaaaaaaaaaaaaaaaaaa"
    }
  },
  "startup_wait": 1.0,
  "docker": {
    "username": "your-username",
    "password": ""
  },
  "redis": {
    "host": "redis",
    "port": 6379
  },
  "forward_stats": true,
  "containers": []
}
```

OK try to launch it!!

```
docker compose up -d
```

To see if it's working, do this:

```docker container ls```

Did a few things start? Are any in a restart loop—if so, something is messed up.

If it looks good, then view the logs:

```docker logs -f deploy-node-1```

It should be moving somewhat quickly, saying "ignoring subscription creation" while it syncs. Scroll to the top to check for errors. If not, then you're good. Wait a few minutes, and once it hits id=2300 or so, it should change like this:

<img width="1132" alt="Screenshot 2024-02-24 at 2 53 14 PM" src="https://github.com/dscoop12345/ritheads/assets/95539970/1adc6fdc-cc81-43a7-b498-d63bdf28a3df">

You can close your Terminal window at any time. You can confirm the node is running by checking the Alchemy dashboard for your Base app, which should show a regular flow of successful requests. Or to be sure you can log back into your server and paste those last two commands. 

## Test the node with an on-chain request

This is [based on these instructions](https://docs.ritual.net/infernet/node/quickstart#run-the-hello-world-container). It costs around $10 in ETH, so make sure your wallet has that much.

Log onto your server with the Terminal app and do `sudo -s`

### Install foundry

```
sudo -s
curl -L https://foundry.paradigm.xyz | bash
```

Then run `foundryup` like it tells you to.

Check if you have 'make' installed by doing `make -version`. If not, do `sudo apt install make`

I also took down my node, just since I was going to be messing with stuff:

```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
```

### Get set up

Clone the files from Ritual

```
cd /
git clone https://github.com/ritual-net/infernet-container-starter
```

Then edit the config file:

```
cd /infernet-container-starter/projects/hello-world/container
nano config.json
```

Change and then save: 
1. the RPC url to your base RPC url
2. the coordinator address to `0x8D871Ef2826ac9001fB2e33fDD6379b6aaBF449c` 
3. the private key to your node wallet's private key
4. the "id" of the first container to something unique. i did "hunty-hello".

Copy everything in this window and save it to paste in the next step.

Go here and paste it:

```
cd /infernet-container-starter/deploy
nano config.json
```

Now deploy it (and your node again):

```
docker compose pull
docker compose up -d
```

Check `docker container ls` to make sure they are running, and `docker logs -f deploy-node-1` to make sure it's not erroring. If it's syncing, give it some time.

### Edit the contract

Do this:

```
cd /infernet-container-starter/projects/hello-world/contracts
forge install foundry-rs/forge-std --no-commit
forge install https://github.com/ritual-net/infernet-sdk --no-commit
```

Then edit the contract:

```
cd src
nano SaysGM.sol
```

Change the "hello-world" line to whatever you named your container. example:

```
solidity
function sayGM() public {
        _requestCompute(
            "hunty-hello",
            bytes("Good morning!"),
            20 gwei,
            1_000_000,
            1
        );
    }
```

And edit the Makefile:

```
cd /infernet-container-starter/projects/hello-world/contracts
nano Makefile
```

You want to replace this stuff with the private key of your wallet and your RPC.

```
sender := 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
RPC_URL := http://localhost:8545
```

Then you edit the deploy script:

```
cd /infernet-container-starter/projects/hello-world/contracts/script
nano Deploy.s.sol
```

And edit it so this line: 
`address coordinator = 0x5FbDB2315678afecb367f032d93F642f64180aa3;`
has this address instead:
`0x8D871Ef2826ac9001fB2e33fDD6379b6aaBF449c`

### Deploy the contract

OK:

```
cd /infernet-container-starter
project=hello-world make deploy-contracts
```

This should tell you a contract address. Copy it, and then go here:

```
cd /infernet-container-starter/projects/hello-world/contracts
cd script
nano CallContract.s.sol
```

And edit it so this line: 
`SaysGM saysGm = SaysGM(0x663F3ad617193148711d28f5334eE4Ed07016602);` 
Has your contract address: 
`SaysGM saysGm = SaysGM(yourContractAddressHereBruv)`

Double check that the hello-world project and node are both running normally in `docker container ls`. Also, double-check that your node is synced by running `docker logs -f deploy-node-1`.

If so, it's time to call the contract!

```
cd /infernet-container-starter
project=hello-world make call-contract
```

If it seems fine, do `docker logs -f deploy-node-1` in another terminal window to see if there are errors or successes.

Check the basescan for your wallet, and you should have a tx saying you 'delivered compute'. Wowie!!!! If not, check the logs for clues what went wrong.

### Do offchain computer

Run this in another Terminal window on your server, using the name you chose earlier instead of hello-world:

```
curl -X POST http://127.0.0.1:4000/api/jobs -H "Content-Type: application/json" -d '{"containers":["darpa-chief-cdrom"], "data": {"some": "input"}}'

```

Some numbers should come out.

Rest.
