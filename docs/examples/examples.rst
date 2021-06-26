Examples
********

Example using a Ledger Nano S::

    ./hwi.py enumerate
    [{"type": "ledger", "path": "IOService:/AppleACPIPlatformExpert/PCI0@0/AppleACPIPCI/XHC1@14/XHC1@14000000/HS02@14200000/Nano S@14200000/Nano S@0/IOUSBHostHIDDevice@14200000,0", "serial_number": "0001"}, {"type": "ledger", "path": "IOService:/AppleACPIPlatformExpert/PCI0@0/AppleACPIPCI/XHC1@14/XHC1@14000000/HS02@14200000/Nano S@14200000/Nano S@1/IOUSBHostHIDDevice@14200000,1", "serial_number": "0001"}]

The OS in this case is macOS v10.13.6  (Darwin Kernel Version 17.7.0). In Linux the
"path" is shorter.

Extracting xpubs
================

Dogecoin Core v1.21.0 and later allows you to retrieve the unspent transaction outputs (utxo)
relevant for a set of `Output Descriptors <https://github.com/dogecoin/dogecoin/blob/master/doc/descriptors.md>`_ with the ``scantxoutset`` RPC call.

To retrieve the outputs relevant for a specific hardware wallet it is
necessary:

1. to derive the xpub of the hardware wallet until the last hardened level
   with HWI (because the private key is required)
2. to use the obtained xpub to compose the output descriptor

These are some schemas used in hardware wallets, with the data necessary to
build the appropriate output descriptor:

+-------------+---------------+--------------------+-------------+
| Used schema | hardened path | further derivation | Output type |
+=============+===============+====================+=============+
| BIP44       | m/44h/3h/0h   | /0/* and /1/*      | pkh()       |
+-------------+---------------+--------------------+-------------+

NOTE:

1. We could also use "combo()" in all cases as "Output Type" because it is a
   "bundle" which includes pk(KEY) and pkh(KEY). If the key is compressed, it
   also includes wpkh(KEY) and sh(wpkh(KEY)).

2. It is possible to specify how many outputs to search for by setting the
   maximum index of the derivation with the "range" key. In the examples
   it is set to 100.

3. The search returns zero outputs (the hardware wallet is empty).

`BIP44 <https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki>`_
-------------------------------------------------------------------------

1. To obtain the xpub relative to the last hardened level (m/44h/3h/0h)

::

    ./hwi.py -t "ledger" -d "IOService:/AppleACPIPlatformExpert/PCI0@0/AppleACPIPCI/XHC1@14/XHC1@14000000/HS02@14200000/Nano S@14200000/Nano S@0/IOUSBHostHIDDevice@14200000,0" getxpub  m/44h/3h/0h
    => b'e0c4000000'
    <= b'1b30010208010003'9000
    => b'f026000000'
    <= b''6d00
    => b'e04000000d038000002c8000000080000000'
    <= b'4104f4b866b49fb76529a076a1c5b25216c1f4b970cb8e3db9874beb15c5371fdb93747fde522d63be4a564dcda8a71c889f5165eac2990cafee9d416141ae8b09c722313667774c7a76697157783146317a653365676850464d58655438666a57466f4b66f9a82310c4530360ec3fee42049fbb7a3c0bfa72fdf2c5b25b09f1c3df21c938'9000
    => b'e040000009028000002c80000000'
    <= b'4104280c846650d7771396a679a55b30c558501f0b5554160c1fbd1d7301c845dacc10c256af2c8d6a13ae4a83763fa747c0d4c09cfa60bfc16714e10b0a938a4a6a2231485451557a6535486571334872553755435174564652745a535839615352674a65d62f97789c088a0b0c3ed57754f75273c6696c0d7812c702ca4f2f72c8631c04'9000
    {"xpub": "xpub6CyidiQae2HF71YigFJqteLsRi9D1EvZJm1Lr4DWWxFVruf3vDSbfyxD9znqVkUTUzc4EdgxDRoHXn64gMbFXQGKXg5nPNfvyVcpuPNn92n"}

2. With this xpub it is possible  extract the relevant UTXOs using the
``scantxoutset`` RPC call in Dogecoin Core v1.21.0.

::

    dogecoin-cli scantxoutset start '[{"desc":"pkh(xpub6CyidiQae2HF71YigFJqteLsRi9D1EvZJm1Lr4DWWxFVruf3vDSbfyxD9znqVkUTUzc4EdgxDRoHXn64gMbFXQGKXg5nPNfvyVcpuPNn92n/0/*)","range":100},
     {"desc":"pkh(xpub6CyidiQae2HF71YigFJqteLsRi9D1EvZJm1Lr4DWWxFVruf3vDSbfyxD9znqVkUTUzc4EdgxDRoHXn64gMbFXQGKXg5nPNfvyVcpuPNn92n/1/*)","range":100}]'
    {
      "success": true,
      "searched_items": 49507771,
      "unspents": [
      ],
      "total_amount": 0.00000000
    }

Binary format handling
======================

The input and output format supported by HWI is base64, which is prescribed by BIP174 as the string format. Note that the PSBT standard also allows for binary formatting when stored as a file. There is no direct support within HWI, but this can be easily accomplished using common utilities. A bash command-line example is detailed below, where the PSBT binary file is stored in ``example.psbt`` and only the common utilities ``base64`` and ``jq`` are required:

::

    cat example.psbt | base64 --wrap=0 | ./hwi.py -t ledger --stdin signtx | jq .[] --raw-output | base64 -d > example_result.psbt
