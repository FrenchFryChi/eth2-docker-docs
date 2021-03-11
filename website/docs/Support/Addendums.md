---
id: Addendums
title:  Addendums.
sidebar_label: Addendums
---

## Addendum: Update the software

This project does not monitor client versions. It is up to you to decide that you
are going to update a component. When you are ready to do so, the below instructions
show you how to.

You can find the current version of your client by running `sudo docker-compose run beacon --version`.


### The eth2-docker tool itself

Inside the project directory, run:<br />
`git pull`

Then `cp .env .env.bak` and `cp default.env .env`, and set variables inside `.env`
the way you need them, with `.env.bak` as a guide.

### The client "stack"
The client "stack" includes (1) Eth 1; (2) Validator Client; (3) Beacon. (idk if this is correct/complete)

If you are using binary build files - the default - you can update everything
in the client "stack" with `sudo docker-compose build --pull`. If you
run shared components in a different directory, such as eth1, traefik, or portainer,
you'd `cd` into those directories and run the above command there.

### Eth1

(add context? why would you use this command if this is about updating?) Run:<br />
`sudo docker-compose build --no-cache --pull eth1`

Then stop, remove and start eth1:<br />
`sudo docker-compose stop eth1 && sudo docker-compose rm eth1`<br />
`sudo docker-compose up -d eth1`

### Client

Beacon and validator client share the same image for most clients, when updating ______ we only need to rebuild one image to capture the updated change in the beacon and validator.

Run:<br />
`sudo docker-compose build --no-cache --pull beacon`

For Prysm, also run:<br />
`sudo docker-compose build --no-cache --pull validator`

And if using the Prysm slasher, run:<br />
`sudo docker-compose build --no-cache --pull slasher`

(does this command apply to all clients, or just Prysm/Prysm slasher? its a bit unclear from the above sequence)Then restart the client:<br />
`sudo docker-compose down && sudo docker-compose up -d eth2`

If you did not store the wallet password with the validator client, come up 
[more manually](#start-the-client) instead. (I don't think this link works, and I don't know what this sentance means either way?)

## Addendum: Remove all traces of the client

This project uses docker volumes to store the Ethereum 1 and Ethereum 2.0 databases, as
well as validator keys for the validator client. You can see these volumes with
`sudo docker volume ls` and remove them with `sudo docker volume rm`, as long as they are
not in use. (Does 'as long as they are not in use' refer to both commands or just removal?)

This can be useful (what can be useful? removal or seeing? or both?) when moving between testnets or from a testnet to main net, without
changing the directory name the project is stored in; or when you want to start over
with a fresh database. Keep in mind that synchronizing Ethereum 1 can take days on main
net, however.

> **Caution** If you are removing the client to recreate it, please be careful
> to wait 10 minutes before importing validator key(s) and starting it again.
> The slashing protection DB will be gone, and you risk slashing your validator(s)
> if you accidentally have a duplicate of your client with the same validator keys.

## Addendum: Add or recover validators

You can use eth2.0-deposit-cli to either recover validator signing keys or add
additional ones, if you wish to deposit more validators against the same mnemonic.
> The same cautions apply as when creating keys in the first place. You
> may wish (and we recommend out of an abundance of caution) to take these steps on a machine that is disconnected from Internet
> and will be wiped immediately after creating the keys. See here(link) for a brief tutorial: https://www.youtube.com/watch?v=oDELXYNSS5w&t=256s 

In the event you need to recover all your validator signing keys, run `sudo docker-compose run --rm deposit-cli-add-recover`(in which directory? eth2.0-deposit-cli?)
and provide your mnemonic, then set index to "0" and the number of validators to the number you had created previously
and are now recreating.

In order to add additional validators and create their signing keys (each additional validator will require 32 ETH), likewise run `sudo docker-compose run --rm deposit-cli-add-recover`
and provide your mnemonic, but this time set the index to the number of validator keys you had created previously,
for example, `4`. New validators will be created after this point. After entering the index number, you will be prompted to enter the number of additional validators you wish to create, for example `2`. You will receive new `keystore-m` signing keys
and a new `deposit_data` JSON for each additional validator you create.

> Please triple-check your work here. You want to be sure the new validator keys are created after (need to elaborate what you mean by after)
> the existing ones. Put another way, you need to ensure that the keys you just created are in fact new, and do not correspond to your already attesting validators. Launchpad will likely safeguard you against depositing twice, but don't rely
> on it. Verify that the public keys in `deposit_data` are new and you did not deposit for them
> previously.
 
## Addendum: Voluntary client exit

Ethereum 2.0 has a concept of "voluntary client exit", which will remove the
validator from attesting duties. Accordingly when you voluntarily exit, your ETH will no longer be attesting, or earning any staking fees, but it will also eliminate the risk of slashing. However, Eth will remain locked in the deposit contract. Locked Eth will be able to be withdrawn after the "merge"
of Ethereum 2.0 with Ethereum 1, but not sooner.

Currently, Prysm and Lighthouse support voluntary exit. This requires a fully synced
beacon node.

### Prysm

To exit using the Prysm client, run `sudo docker-compose run --rm validator-voluntary-exit` and follow the
prompts.

If you wish to exit validators that were running on other clients, you can do this
as follows:

- Stop the other client(s), and wait 10 minutes. This is so you won't have
  a validator attest in the same epoch twice.
- Copy all `keystore-m` JSON files into `.eth2/validator_keys` in this project
  directory.
- Stop the Prysm client in this project, `sudo docker-compose down`
- Import the new keys via `sudo docker-compose run validator-import`. Note
  that Prysm assumes they all have the same password. If that's not the case,
  maybe work in batches.
- Verify once more that the old client is down, has been for 10 minutes, and
  can't come back up. **If both the old client and this Prysm run at the same time,
  you will slash yourself**
- Bring the Prysm client up: `sudo docker-compose up -d eth2`
- Check logs until the beacon is synced: `sudo docker-compose logs -f beacon`
- Initiate voluntary exit and follow the prompts: `sudo docker-compose run --rm validator-voluntary-exit`


### Lighthouse

The exit procedure for lighthouse is not very refined, yet.

- Copy the `keystore-m` JSON files into `.eth2/validator_keys` in this project
  directory.
- Run `sudo docker-compose run --rm validator-voluntary-exit /var/lib/lighthouse/validator_keys/<name-of-keystore-file>`,
  once for each keystore (validator) you wish to exit.
- Follow prompts.

> The directory `.eth2/validator_keys` is passed through to docker as `/var/lib/lighthouse/validator_keys`. Lighthouse
> expects you to explicitly name the `keystore-m` file for which you wish to process an exit to ensure that you are exiting only the validator(s) you intend. Because this can
> be confusing, here's an example:
```
yorick@ethlinux:~/eth2-pyrmont$ ls .eth2/validator_keys/
deposit_data-1605672506.json  keystore-m_12381_3600_0_0_0-1605672506.json
yorick@ethlinux:~/eth2-pyrmont$ sudo docker-compose run --rm validator-voluntary-exit /var/lib/lighthouse/validator_keys/keystore-m_12381_3600_0_0_0-1605672506.json
Starting eth2-pyrmont_beacon_1 ... done
Running account manager for pyrmont testnet
validator-dir path: "/var/lib/lighthouse/validators" 
Enter the keystore password for validator in "/var/lib/lighthouse/validator_keys/keystore-m_12381_3600_0_0_0-1605672506.json"  
``` 
^This whole thing was confusing and lacked context as far as variables, but a more experienced user might understand.

### Nimbus

You will need to know the index or public key of the validator for which you intend to voluntarily exit as it shows on https://beaconcha.in/ or https://pyrmont.beaconcha.in/ if on Pyrmont testnet.

Run `sudo docker-compose run --rm validator-voluntary-exit <INDEX or 0xPUBKEY>` and follow prompts to exit. For example:
- If using an index, here 0, `sudo docker-compose run --rm validator-voluntary-exit 0`
- If using a public key, you need to include "0x" in front of it, for example `sudo docker-compose run --rm validator-voluntary-exit 0xb0127e191555550fae82788061320428d2cef31b0807aa33b88f48c53682baddce6398bb737b1ba5c503ca696d0cab4a`

### Avoid penalties

Note you will need to continue running your validator until the exit
has been processed by the chain, if you wish to avoid incurring offline
penalties. You can check the status of your validator with tools such
as [beaconcha.in](https://beaconcha.in) and [beaconscan](https://beaconscan.com).

## Addendum: Moving a client

Please see the [moving checklist](../Support/Moving.md)

## Addendum: Troubleshooting

A few useful commands if you run into issues. As always, `sudo` is a Linux-ism a that indicates to you are intending to act as an admin user, without `sudo` otherwise correct commands may not work. `Sudo` may not be needed on MacOS.

`sudo docker-compose stop servicename` brings a service down, for example `docker-compose stop validator`.<br />
`sudo docker-compose down` will stop the entire stack.<br />
`sudo docker-compose up -d servicename` starts a single service, for example `sudo docker-compose up -d validator`.
The `-d` means "detached", not connected to your input session.<br />
`sudo docker-compose run servicename` starts a single service and connects your input session to it. Use the Ctrl-p Ctrl-q
key sequence to detach from it again.
`sudo docker-compose up -d eth2` will start the entire stack. (not sure if its eth2 for everyone or just me)

`sudo docker ps` lists all running services, with the container name to the right.<br />
`sudo docker logs containername` shows logs for a container, `sudo docker logs -f containername` scrolls them.<br />
`sudo docker-compose logs servicename` shows logs for a service, `sudo docker-compose logs -f servicename` scrolls them.<br />
`sudo docker exec -it containername /bin/bash` will connect you to a running service in a bash shell. The eth1 service doesn't have a shell
if using geth.<br />

You may start a service with `sudo docker-compose up -d servicename` and then find it's not in `sudo docker ps`. That means it terminated while
trying to start. To investigate, you could leave the `-d` off so you see logs on command line:<br />
`sudo docker-compose up beacon`, for example.<br />
You could also run `sudo docker-compose logs beacon` to see the last logs of that service and the reason it terminated.<br />

If a service is not starting and you want to bring up its container manually, so you can investigate, first bring everything down:<br />
`sudo docker-compose down`, tear down everything first.<br />
`sudo docker ps`, make sure everything is down.<br />

If you need to see the files that are being stored by services such as beacon, validator, eth1 node, grafana, &c, in Ubuntu Linux you can find
those in /var/lib/docker/volumes. `sudo bash` to invoke a shell that has access.

**HERE BE DRAGONS** You can totally run N copies of an image manually and then successfully start a validator in each and get yourself slashed.
Take extreme care. (I think this could use some elaboration, even if just a sentance or two giving an example and how to avoid).

Once your stack is down, to run an image and get into a shell, without executing the client automatically:<br />
`sudo docker run -it --entrypoint=/bin/bash imagename`, for example `sudo docker run -it --entrypoint=/bin/bash lighthouse`.<br />
You'd then run Linux commands manually in there, you could start components of the client manually. There is one image per client.<br />
`sudo docker images` will show you all images.

## Addendum: Additional resources

[Youtube Channel](https://www.youtube.com/c/YorickDowne)
[Security Best Practices](https://www.coincashew.com/coins/overview-eth/guide-or-security-best-practices-for-a-eth2-validator-beaconchain-node) and (https://www.youtube.com/watch?v=hHtvCGlPz-o&t=1s)

# Guiding principles:

- Reduce the attack surface of the client as much as feasible.
  None of the eth2 clients lend themselves to be statically compiled and running
  in "scratch" containers, alas.
- Guide users to good key management as much as possible (which keys? ssh? validator keys? mnemonics? all of the above?)
- Create something that makes for a good user experience and guides people new to docker and Linux as much as feasible
