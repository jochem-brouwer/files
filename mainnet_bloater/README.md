The payloads here are built on mainnet @ 24350000.
It includes this prestate:
- EOA from 0x00..001000 to 0x00..001000 + 150_000 (these are enough accounts to target for gas limits up to 300M)
- A 10gb storage contract at 0x87a6314da5ac8832f6e7a176c8fb133b19f5be04 with 0x140002fa slots occupied (multiplied by 32 is this >10gb). This address is an EOA.
    - The code can be overwritten using EIP-7702 (this keeps the 10Gb storage). The pkey is 0x4da32d29f6dcffa26e09dc4e102033f2d105de1444fb893493ae703289275e0e
 
Deploy from CREATE2 factory 0x4e59b44847b379578588920ca78fbf26c0b4956c 

Initcode:
0x7f5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b6000526020600060205e6040600060405e6080600060805e61010060006101005e61020060006102005e61040060006104005e61080060006108005e61100060006110005e61200060006120005e61400060006140005e7f615fff565b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b5b6000527f5b5b5b5b5b5b5b5b5b5b5b5b000000000000000000000000000000000000000030176020526160006000f3

This generates a PUSH 0x5FFF JUMP JUMPDEST JUMPDEST ... JUMPDEST code, where in part of the code the address of the contract is inserted to guarnatee uniqueness.
It creates the max sized contract for Osaka (24 KiB)
