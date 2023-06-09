# 7.2: Using a Partially Signed Bitcoin Transaction

> :information_source: **NOTE:** This section has been recently added to the course and is an early draft that may still be awaiting review. Caveat reader.

Now that you've learned the basic workflow of generating a PSBT, you probably want to do something with it. What can PSBTs do that multi-sigs (and normal raw transactions) can't? To start with, you've got the ease of use of a standardized format, which means that you can use your `bitcoin-cli` transactions and meld them with transactions generated by people (or programs) on other platforms. Beyond that, you can do some things that just weren't easy using other mechanics.

Following are three examples of using PSBTs for: multi-sigs, pooling money, and joining coins.

> :warning: **VERSION WARNING:** This is an innovation from Bitcoin Core v 0.17.0. Earlier versions of Bitcoin Core will not be able to work with the PSBT while it is in progress (though they will still be able to recognize the final transaction).

## Use a PSBT to Spend MultiSig Funds

Assume you've created a multisig address, just like you did in [§6.3](06_3_Sending_an_Automated_Multisig.md). 
```
machine1$ bitcoin-cli -named addmultisigaddress nrequired=2 keys='''["'$pubkey1'","'$pubkey2'"]'''
{
  "address": "tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0",
  "redeemScript": "5221038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e2103789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd063652ae",
  "descriptor": "wsh(multi(2,[d6043800/0'/0'/26']038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e,[be686772]03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636))#07zyayfk"
}
machine1$ bitcoin-cli -named importaddress address="tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0" rescan=false

machine2$ bitcoin-cli -named addmultisigaddress nrequired=2 keys='''["'$pubkey1'","'$pubkey2'"]'''
{
  "address": "tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0",
  "redeemScript": "5221038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e2103789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd063652ae",
  "descriptor": "wsh(multi(2,[d6043800/0'/0'/26']038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e,[be686772]03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636))#07zyayfk"
}
machine2$ bitcoin-cli -named importaddress address="tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0" rescan=false
```
And, you've got some money in it:
```
$ bitcoin-cli listunspent
[
  {
    "txid": "53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5",
    "vout": 0,
    "address": "tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0",
    "label": "",
    "witnessScript": "5221038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e2103789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd063652ae",
    "scriptPubKey": "0020224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
    "amount": 0.01999800,
    "confirmations": 2,
    "spendable": false,
    "solvable": true,
    "desc": "wsh(multi(2,[d6043800/0'/0'/26']038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e,[be686772]03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636))#07zyayfk",
    "safe": true
  }
]
```
You _could_ spend this using the mechanisms in [Chapter 6](06_0_Expanding_Bitcoin_Transactions_Multisigs.md), where you serially signed a transaction, but instead we're going to show the advantage of PSBTs for multi-sigs: you can generate a single PSBT, allow everyone to sign that in parallel, and then combine the results! There's no more laboriously passing an ever-expanding hex from person to person, which speeds things up and reduces the chances of errors.

To demonstrate this methodology, we're going to pull that 0.02 BTC out of the multi-sig and divide it between the two signers, who each generated a new address for that purpose:
```
machine1$ bitcoin-cli getnewaddress
tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma
machine2$ bitcoin-cli getnewaddress
tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp
```
The first thing we do is create a PSBT on the machine of our choice. (It doesn't matter which.) We need to use `createpsbt` from [§7.1](07_1_Creating_a_Partially_Signed_Bitcoin_Transaction.md) for this, not the simpler `walletcreatefundedpsbt`, because we need the extra control of selecting the money protected by the multi-sig. (This will be the case for all three examples in this section, which demonstrates why you usually need to use `createpsbt` for the complex stuff.)
```
machine1$ utxo_txid=53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5
machine1$ utxo_vout=0
machine1$ split1=tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma
machine1$ split2=tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp
machine1$ psbt=$(bitcoin-cli -named createpsbt inputs='''[ { "txid": "'$utxo_txid'", "vout": '$utxo_vout' } ]''' outputs='''{ "'$split1'": 0.009998,"'$split2'": 0.009998 }''')
```
You then need to send that $psbt to everyone for signing:
```
machine1$ echo $psbt
cHNidP8BAHECAAAAAbU5tQSXtwlf5ZamU+wwrLjHFp1p6WQh7haL/sLFYuxTAAAAAAD/////AnhBDwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDad4QQ8AAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAAAAAA=
```
But you just have to send it once! And you do it simulataneously.

Here's the result on the first machine, where we generated the PSBT:
```
machine1$ psbt_p1=$(bitcoin-cli walletprocesspsbt $psbt | jq -r '.psbt')
machine1$ bitcoin-cli decodepsbt $psbt_p1
{
  "tx": {
    "txid": "1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985",
    "hash": "1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985",
    "version": 2,
    "size": 113,
    "vsize": 113,
    "weight": 452,
    "locktime": 0,
    "vin": [
      {
        "txid": "25e8a26f60cf485768a1e6953b983675c867b7ab126b02e753c47b7db0c4be5e",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      }
    ],
    "vout": [
      {
        "value": 0.00499900,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 cee9f88288a5f4980191a28d7ae08ff584050da7",
          "hex": "0014cee9f88288a5f4980191a28d7ae08ff584050da7",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma"
          ]
        }
      },
      {
        "value": 0.00049990,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 8d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "hex": "00148d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp"
          ]
        }
      }
    ]
  },
  "unknown": {
  },
  "inputs": [
    {
      "witness_utxo": {
        "amount": 0.01000000,
        "scriptPubKey": {
          "asm": "0 2abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
          "hex": "00202abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
          "type": "witness_v0_scripthash",
          "address": "tb1q92a46jww0e6ne066nlagekhczkl3qa84czl5jk5nm78t2yf0vk4qte328m"
        }
      },
      "partial_signatures": {
        "03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d": "304402203abb95d1965e4cea630a8b4890456d56698ff2dd5544cb79303cc28cb011cbb40220701faa927f8a19ca79b09d35c78d8d0a2187872117d9308805f7a896b07733f901"
      },
      "witness_script": {
        "asm": "2 033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0 03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d 2 OP_CHECKMULTISIG",
        "hex": "5221033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa02103f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d52ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0",
          "master_fingerprint": "c1fdfe64",
          "path": "m"
{
  "tx": {
    "txid": "ee82d3e0d225e0fb919130d68c5052b6e3c362c866acc54d89af975330bb4d16",
    "hash": "ee82d3e0d225e0fb919130d68c5052b6e3c362c866acc54d89af975330bb4d16",
    "version": 2,
    "size": 113,
    "vsize": 113,
    "weight": 452,
    "locktime": 0,
    "vin": [
      {
        "txid": "53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      }
    ],
    "vout": [
      {
        "value": 0.00999800,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 cee9f88288a5f4980191a28d7ae08ff584050da7",
          "hex": "0014cee9f88288a5f4980191a28d7ae08ff584050da7",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma"
          ]
        }
      },
      {
        "value": 0.00999800,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 8d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "hex": "00148d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp"
          ]
        }
      }
    ]
  },
  "unknown": {
  },
  "inputs": [
    {
      "witness_utxo": {
        "amount": 0.01999800,
        "scriptPubKey": {
          "asm": "0 224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
          "hex": "0020224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
          "type": "witness_v0_scripthash",
          "address": "tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0"
        }
      },
      "partial_signatures": {
        "038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e": "3044022040aae4f2ba37b1526524195f4a325d97d1317227b3c82aea55c5abd66810a7ec0220416e7c03e70a31232044addba454d6b37b6ace39ab163315d3293e343ae9513301"
      },
      "witness_script": {
        "asm": "2 038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e 03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636 2 OP_CHECKMULTISIG",
        "hex": "5221038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e2103789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd063652ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636",
          "master_fingerprint": "be686772",
          "path": "m"
        },
        {
          "pubkey": "038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e",
          "master_fingerprint": "d6043800",
          "path": "m/0'/0'/26'"
        }
      ]
    }
  ],
  "outputs": [
    {
      "bip32_derivs": [
        {
          "pubkey": "02fce26085452d07abc63bd389cb7dba9871e79bbecd08039291226be8232a9000",
          "master_fingerprint": "d6043800",
          "path": "m/0'/0'/24'"
        }
      ]
    },
    {
    }
  ],
  "fee": 0.00000200
}
machine1$ bitcoin-cli analyzepsbt $psbt_p1
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "signer",
      "missing": {
        "signatures": [
          "be6867729bcc35ed065bb4c937557d371218a8e2"
        ]
      }
    }
  ],
  "estimated_vsize": 168,
  "estimated_feerate": 0.00001190,
  "fee": 0.00000200,
  "next": "signer"
}

```
This demonstrates that the UTXO information has been imported, and that there's a _partial signature_, but that the signing of the single input is still not complete.

Here's the same thing on the other machine:
```
machine2$ psbt=cHNidP8BAHECAAAAAbU5tQSXtwlf5ZamU+wwrLjHFp1p6WQh7haL/sLFYuxTAAAAAAD/////AnhBDwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDad4QQ8AAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAAAAAA=
machine2$ psbt_p2=$(bitcoin-cli walletprocesspsbt $psbt | jq -r '.psbt')
machine3$ echo $psbt_p2
cHNidP8BAHECAAAAAbU5tQSXtwlf5ZamU+wwrLjHFp1p6WQh7haL/sLFYuxTAAAAAAD/////AnhBDwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDad4QQ8AAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAABASu4gx4AAAAAACIAICJMtQOn94NXmbnCLuDDx9k9CQNW4w5wAVw+u/pRWjB0IgIDeJ9UNCNnDhaWZ/9+Hy2iqX3xsJEicuFC1YJFGs69BjZHMEQCIDJ71isvR2We6ym1QByLV5SQ+XEJD0SAP76fe1JU5PZ/AiB3V7ejl2H+9LLS6ubqYr/bSKfRfEqrp2FCMISjrWGZ6QEBBUdSIQONc63yx+oz+dw0t3titZr0M8HenHYzMtp56D4VX5YDDiEDeJ9UNCNnDhaWZ/9+Hy2iqX3xsJEicuFC1YJFGs69BjZSriIGA3ifVDQjZw4Wlmf/fh8toql98bCRInLhQtWCRRrOvQY2ENPtiCUAAACAAAAAgAYAAIAiBgONc63yx+oz+dw0t3titZr0M8HenHYzMtp56D4VX5YDDgRZu4lPAAAiAgNJzEMyT3rZS7QHqb8SvFCv2ee0MKRyVy8bY8tVUDT1KhDT7YglAAAAgAAAAIADAACAAA==
```
Note again that we managed the signing of this multi-sig by generating a totally unsigned PSBT with the correct UTXO, then allowing each of the users to process that PSBT on their own, adding inputs and signatures. As a result, we have two PSBTs each of which contain one signature and not the other. That wouldn't work in the classic multi-sig scenario, because all the signatures have to be serialized. Here, instead, we can sign in parallel and then make use of the Combiner role to mush those together.

We again go to either machine, and make sure we have both PSBTs in variables, then we combine them:
```
machine1$ psbt_p2="cHNidP8BAHECAAAAAbU5tQSXtwlf5ZamU+wwrLjHFp1p6WQh7haL/sLFYuxTAAAAAAD/////AnhBDwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDad4QQ8AAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAABAIcCAAAAAtu5pTheUzdsTaMCEPj3XKboMAyYzABmIIeOWMhbhTYlAAAAAAD//////uSTLbibcqSd/Z9ieSBWJ2psv+9qvoGrzWEa60rCx9cAAAAAAP////8BuIMeAAAAAAAiACAiTLUDp/eDV5m5wi7gw8fZPQkDVuMOcAFcPrv6UVowdAAAAAAAACICA0nMQzJPetlLtAepvxK8UK/Z57QwpHJXLxtjy1VQNPUqENPtiCUAAACAAAAAgAMAAIAA"
machine2$ psbt_c=$(bitcoin-cli combinepsbt '''["'$psbt_p1'", "'$psbt_p2'"]''')
$ bitcoin-cli decodepsbt $psbt_c
{
  "tx": {
    "txid": "ee82d3e0d225e0fb919130d68c5052b6e3c362c866acc54d89af975330bb4d16",
    "hash": "ee82d3e0d225e0fb919130d68c5052b6e3c362c866acc54d89af975330bb4d16",
    "version": 2,
    "size": 113,
    "vsize": 113,
    "weight": 452,
    "locktime": 0,
    "vin": [
      {
        "txid": "53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      }
    ],
    "vout": [
      {
        "value": 0.00999800,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 cee9f88288a5f4980191a28d7ae08ff584050da7",
          "hex": "0014cee9f88288a5f4980191a28d7ae08ff584050da7",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma"
          ]
        }
      },
      {
        "value": 0.00999800,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 8d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "hex": "00148d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp"
          ]
        }
      }
    ]
  },
  "unknown": {
  },
  "inputs": [
    {
      "witness_utxo": {
        "amount": 0.01999800,
        "scriptPubKey": {
          "asm": "0 224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
          "hex": "0020224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
          "type": "witness_v0_scripthash",
          "address": "tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0"
        }
      },
      "partial_signatures": {
        "038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e": "3044022040aae4f2ba37b1526524195f4a325d97d1317227b3c82aea55c5abd66810a7ec0220416e7c03e70a31232044addba454d6b37b6ace39ab163315d3293e343ae9513301",
        "03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636": "30440220327bd62b2f47659eeb29b5401c8b579490f971090f44803fbe9f7b5254e4f67f02207757b7a39761fef4b2d2eae6ea62bfdb48a7d17c4aaba761423084a3ad6199e901"
      },
      "witness_script": {
        "asm": "2 038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e 03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636 2 OP_CHECKMULTISIG",
        "hex": "5221038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e2103789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd063652ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "03789f543423670e169667ff7e1f2da2a97df1b0912272e142d582451acebd0636",
          "master_fingerprint": "be686772",
          "path": "m"
        },
        {
          "pubkey": "038d73adf2c7ea33f9dc34b77b62b59af433c1de9c763332da79e83e155f96030e",
          "master_fingerprint": "d6043800",
          "path": "m/0'/0'/26'"
        }
      ]
    }
  ],
  "outputs": [
    {
      "bip32_derivs": [
        {
          "pubkey": "02fce26085452d07abc63bd389cb7dba9871e79bbecd08039291226be8232a9000",
          "master_fingerprint": "d6043800",
          "path": "m/0'/0'/24'"
        }
      ]
    },
    {
      "bip32_derivs": [
        {
          "pubkey": "0349cc43324f7ad94bb407a9bf12bc50afd9e7b430a472572f1b63cb555034f52a",
          "master_fingerprint": "d3ed8825",
          "path": "m/0'/0'/3'"
        }
      ]
    }
  ],
  "fee": 0.00000200
}
$ bitcoin-cli analyzepsbt $psbt_c
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "finalizer"
    }
  ],
  "estimated_vsize": 168,
  "estimated_feerate": 0.00001190,
  "fee": 0.00000200,
  "next": "finalizer"
}
```
It worked! We just finalize and send and we're done:
```
machine2$ psbt_c_hex=$(bitcoin-cli finalizepsbt $psbt_c | jq -r '.hex')
standup@btctest2:~$ bitcoin-cli -named sendrawtransaction hexstring=$psbt_c_hex
ee82d3e0d225e0fb919130d68c5052b6e3c362c866acc54d89af975330bb4d16
```
Obviously, there wasn't a big improvement in using this method over serially signing a transaction for a 2-of-2 multisig when everyone was using `bitcoin-cli`: we could have passed a raw transaction with partial signatures from one user to the other just as easily as sending that PSBT. But, this was the simplest case. As we delve into more complex multisigs, this methodology becomes better and better. 

First of all, it's platform independent. As long as everyone is using a service that supports Bitcoin Core 0.17, they'll all be able to sign this transaction, which isn't true when classic multi-sigs are being passed around among different platforms. 

Second, it's a lot more scalable. Consider a 3-of-5 multisig. Under the old methodology it would have to passed from person to person, greatly increasing the problems if any single link in the chain breaks. Here, other users just have to send the PSBTs back to the Creator, and as soon as she has enough, she can generate the final transaction.

## Use a PSBT to Pool Money

Multisigs like the one used in the previous example are often used to receive payments for collaborative work, whether it be royalties for a book or payments made to a company. In that situation, the above example works great: the two participants receive their money which they then split up. But what about the converse case, where two (or more) participants want to set up a joint venture, and they need to seed it with money? 

The traditional answer is to create a multisig, then to have the participants individually send their funds to it. The problem is that the first payer has to depend on the good faith of the second, and that doesn't build on the strength of Bitcoin, which is its _trustlessness_. Fortunately, with the advent of PSBTs, we can now make trustless payments that pool funds.

> :book: ***What does trustless mean?*** Trustless means that no participant has to trust any other participant. They instead expect the software protocols to ensure that everything is enacted fairly in an expected manner. Bitcoin is a trustless protocol because you don't need anyone else to act in good faith; the system manages it. Similarly, PSBTs allow for the trustless creation of transactions that pool or split funds.

The following example shows two users who each have 0.010 BTC that they want to pool to the multisig address `tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0`, created above.
```
machine1$ bitcoin-cli listunspent
[
  {
    "txid": "2536855bc8588e87206600cc980c30e8a65cf7f81002a34d6c37535e38a5b9db",
    "vout": 0,
    "address": "tb1qfg5y4fx979xkv4ezatc5eevufc8vh45553n4ut",
    "label": "",
    "scriptPubKey": "00144a284aa4c5f14d665722eaf14ce59c4e0ecbd694",
    "amount": 0.01000000,
    "confirmations": 2,
    "spendable": true,
    "solvable": true,
    "desc": "wpkh([d6043800/0'/0'/25']02bea222cf9ea1f49b392103058cc7c8741d76a553fe627c1c43fc3ef4404c9d54)#4hnkg9ml",
    "safe": true
  }
]
machine2$ bitcoin-cli listunspent
[
 {
    "txid": "d7c7c24aeb1a61cdab81be6aefbf6c6a27562079629ffd9da4729bb82d93e4fe",
    "vout": 0,
    "address": "tb1qfqyyw6xrghm5kcrpkus3kl2l6dz4tpwrvn5ujs",
    "label": "",
    "scriptPubKey": "001448084768c345f74b6061b7211b7d5fd3455585c3",
    "amount": 0.01000000,
    "confirmations": 5363,
    "spendable": true,
    "solvable": true,
    "desc": "wpkh([d3ed8825/0'/0'/0']03ff6b94c119582a63dbae4fb530efab0ed5635f7c3b2cf171264ca0af3ecef33a)#gtmd2e2k",
    "safe": true
  }
]
```
They set up variables to use those transactions:
```
machine1$ utxo_txid_1=2536855bc8588e87206600cc980c30e8a65cf7f81002a34d6c37535e38a5b9db
machine1$ utxo_vout_1=0
machine1$ utxo_txid_2=d7c7c24aeb1a61cdab81be6aefbf6c6a27562079629ffd9da4729bb82d93e4fe
machine1$ utxo_vout_2=0
machine1$ multisig=tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0

```
And create a PSBT:
```
machine1$ psbt=$(bitcoin-cli -named createpsbt inputs='''[ { "txid": "'$utxo_txid_1'", "vout": '$utxo_vout_1' }, { "txid": "'$utxo_txid_2'", "vout": '$utxo_vout_2' } ]''' outputs='''{ "'$multisig'": 0.019998 }''')
```
Here's what it looks like:
```
machine1$ bitcoin-cli decodepsbt $psbt
{
  "tx": {
    "txid": "53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5",
    "hash": "53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5",
    "version": 2,
    "size": 135,
    "vsize": 135,
    "weight": 540,
    "locktime": 0,
    "vin": [
      {
        "txid": "2536855bc8588e87206600cc980c30e8a65cf7f81002a34d6c37535e38a5b9db",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      },
      {
        "txid": "d7c7c24aeb1a61cdab81be6aefbf6c6a27562079629ffd9da4729bb82d93e4fe",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      }
    ],
    "vout": [
      {
        "value": 0.01999800,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
          "hex": "0020224cb503a7f7835799b9c22ee0c3c7d93d090356e30e70015c3ebbfa515a3074",
          "reqSigs": 1,
          "type": "witness_v0_scripthash",
          "addresses": [
            "tb1qyfxt2qa877p40xdecghwps78my7sjq6kuv88qq2u86al5526xp6qfqjud0"
          ]
        }
      }
    ]
  },
  "unknown": {
  },
  "inputs": [
    {
    },
    {
    }
  ],
  "outputs": [
    {
    }
  ]
}
```
It doesn't matter that the transactions are owned by two different people or that their full information appears on two different machines. This funding PSBT will work exactly the same as the multisig PSBT: once all of the controlling parties have signed, then the transaction can be finalized.

Here's the process, this time passing the partially signed PSBT from one user to another rather than having to combine things at the end.
```
machine1$ bitcoin-cli walletprocesspsbt $psbt
{
  "psbt": "cHNidP8BAIcCAAAAAtu5pTheUzdsTaMCEPj3XKboMAyYzABmIIeOWMhbhTYlAAAAAAD//////uSTLbibcqSd/Z9ieSBWJ2psv+9qvoGrzWEa60rCx9cAAAAAAP////8BuIMeAAAAAAAiACAiTLUDp/eDV5m5wi7gw8fZPQkDVuMOcAFcPrv6UVowdAAAAAAAAQEfQEIPAAAAAAAWABRKKEqkxfFNZlci6vFM5ZxODsvWlAEIawJHMEQCIGAiKIAWRXiw68o3pw61/cVNP7n2oH73S654XXgQ4kjHAiBtTBqmaF1iIzYGXrG4DadH8y6mTuCRVFDiPl+TLQDBJwEhAr6iIs+eofSbOSEDBYzHyHQddqVT/mJ8HEP8PvRATJ1UAAABAUdSIQONc63yx+oz+dw0t3titZr0M8HenHYzMtp56D4VX5YDDiEDeJ9UNCNnDhaWZ/9+Hy2iqX3xsJEicuFC1YJFGs69BjZSriICA3ifVDQjZw4Wlmf/fh8toql98bCRInLhQtWCRRrOvQY2BL5oZ3IiAgONc63yx+oz+dw0t3titZr0M8HenHYzMtp56D4VX5YDDhDWBDgAAAAAgAAAAIAaAACAAA==",
  "complete": false
}

machine2$  psbt_p="cHNidP8BAIcCAAAAAtu5pTheUzdsTaMCEPj3XKboMAyYzABmIIeOWMhbhTYlAAAAAAD//////uSTLbibcqSd/Z9ieSBWJ2psv+9qvoGrzWEa60rCx9cAAAAAAP////8BuIMeAAAAAAAiACAiTLUDp/eDV5m5wi7gw8fZPQkDVuMOcAFcPrv6UVowdAAAAAAAAQEfQEIPAAAAAAAWABRKKEqkxfFNZlci6vFM5ZxODsvWlAEIawJHMEQCIGAiKIAWRXiw68o3pw61/cVNP7n2oH73S654XXgQ4kjHAiBtTBqmaF1iIzYGXrG4DadH8y6mTuCRVFDiPl+TLQDBJwEhAr6iIs+eofSbOSEDBYzHyHQddqVT/mJ8HEP8PvRATJ1UAAABAUdSIQONc63yx+oz+dw0t3titZr0M8HenHYzMtp56D4VX5YDDiEDeJ9UNCNnDhaWZ/9+Hy2iqX3xsJEicuFC1YJFGs69BjZSriICA3ifVDQjZw4Wlmf/fh8toql98bCRInLhQtWCRRrOvQY2BL5oZ3IiAgONc63yx+oz+dw0t3titZr0M8HenHYzMtp56D4VX5YDDhDWBDgAAAAAgAAAAIAaAACAAA=="
machine2$ psbt_f=$(bitcoin-cli walletprocesspsbt $psbt_p | jq -r '.psbt')
machine2$ bitcoin-cli analyzepsbt $psbt_f
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": true,
      "next": "extractor"
    },
    {
      "has_utxo": true,
      "is_final": true,
      "next": "extractor"
    }
  ],
  "estimated_vsize": 189,
  "estimated_feerate": 0.00001058,
  "fee": 0.00000200,
  "next": "extractor"
}
machine2$ psbt_hex=$(bitcoin-cli finalizepsbt $psbt_f | jq -r '.hex')
machine2$ bitcoin-cli -named sendrawtransaction hexstring=$psbt_hex
53ec62c5c2fe8b16ee2164e9699d16c7b8ac30ec53a696e55f09b79704b539b5
```
We've used a PSBT to trustlessly gather money into a multisig!

## Use a PSBT to CoinJoin

CoinJoin is another Bitcoin application that requires trustlessness. Here, you have a variety of parties who don't necessarily know each other joining money and getting it back.

The methodology for managing it with PSBTs is exactly the same as you've seen in the above examples, as the following pseudo-code demonstrates:
```
$ psbt=$(bitcoin-cli -named createpsbt inputs='''[ { "txid": "'$utxo_txid_1'", "vout": '$utxo_vout_1' }, { "txid": "'$utxo_txid_2'", "vout": '$utxo_vout_2' }, { "txid": "'$utxo_txid_3'", "vout": '$utxo_vout_3' } ]''' outputs='''{ "'$split1'": 1.7,"'$split2'": 0.93,"'$split3'": 1.4 }''')
```
Each user puts in their own UTXO, and each one receives a corresponding output. 

The best way to manage a CoinJoin is to send out the base PSBT to all the parties (who could be numerous), and then have them each sign the PSBT and send back to a single party who will combine, finalize, and send.

> :book: ***What is CoinJoin?*** CoinJoin is a methodology whereby a group of people can mix together their cryptocurrency, helping to reduce fungibility for all the coins. Each person puts in and takes out the same amount of coins (minus transaction fees) in a multi-person transaction that is simultaneously conducted by a _large_ number of people.  It's designed to be "trustless" so that the parties don't need to know or trust each other. A CoinJoin ultimately increases anonymity by making the coins hard to trace. The Wasabi Wallet and a number of "mixer" services support CoinJoin at the large-scale necessary for improved anonymity.

## Summary: Using a Partially Signed Bitcoin Transaction

You've now seen the PSBT process that you learned in [§7.1](07_1_Creating_a_Partially_Signed_Bitcoin_Transaction.md) in use in three real-life examples: creating a multi-sig, pooling funds, and CoinJoining. These were all theoretically possible in classic Bitcoin by having multiple people sign carefully constructed transactions, but PSBTs make it standardized and simple.

> :fire: ***What's the power of a PSBT?*** A PSBT allows for the creation of trustless transactions between multiple parties and multiple machines. If more than one party would need to fund a transaction, if more than one party would need to sign a transaction, or if a transaction needs to be created on one machine and signed on another, then a PSBT makes it simple without depending on the non-standardized partial signing mechanisms that used to exist before PSBT.

That last point, on creating a transaction on one machine and signing on another, is an element of PSBTs that we haven't gotten to yet. It's at the heart of hardware wallets, where you often want to create a transaction on a full node, then pass it on to a hardware wallet when a signature is required. That's the topic of the last section (and our fourth real-life example) in this chapter on PSBTs.

## What's Next?

Continue "Expanding Bitcoin Transactions with PSBTs" with [§7.3: Integrating with Hardware Wallets](07_3_Integrating_with_Hardware_Wallets.md).
