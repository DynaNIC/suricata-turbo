# Suricata DynaNIC demo

## Compile instructions

1) Install minimal/recommended Suricata dependencies
1) Install DynaNIC package (with colocated DPDK)
1) Configure and compile Suricata using `./autogen.sh && ./configure --enable-dpdk CFLAGS="-g -O3" && make -j10`
1) Verify that Suricata compiled with e.g.: `./src/suricata -V`

## Run instructions

1) Enable DynaNIC `nfb-eth -e1`
1) Allocate hugepages `sudo dpdk-hugepages.py --setup 2G`
1) Adjust to real PCIe address in the Suricata configuration file (`./demo/suricata.yaml.ndp.16thr.demo.staticdynamic`) -- leave `_eth0` suffix.
1) Adjust `drop-filter` rules in the Suricata configuration file if necessary.
1) Run Suricata `sudo ./src/suricata -c ./demo/suricata.yaml.ndp.16thr.demo.staticdynamic -l /tmp/ -S /dev/null -vvvv --dpdk`

