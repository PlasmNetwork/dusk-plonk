# PLONK
![Build Status](https://github.com/dusk-network/plonk/workflows/Continuous%20integration/badge.svg)
[![Repository](https://img.shields.io/badge/github-plonk-blueviolet?logo=github)](https://github.com/dusk-network/plonk)
[![Documentation](https://img.shields.io/badge/docs-plonk-blue?logo=rust)](https://docs.rs/plonk/)


_This is a pure Rust implementation of the PLONK proving system over BLS12-381_

This library contains a modularised implementation of KZG10 as the default polynomial commitment scheme.

## Usage

```rust
use dusk_plonk::prelude::*;
use rand::SeedableRng;
use rand_xorshift::XorShiftRng;
use rand_core::RngCore;

// Implement a circuit that checks:
// 1) a + b = c where C is a PI
// 2) a <= 2^6
// 3) b <= 2^5
// 4) a * b = d where D is a PI
// 5) JubJub::GENERATOR * e(JubJubScalar) = f where F is a Public Input
#[derive(Debug, Default)]
pub struct TestCircuit {
    a: BlsScalar,
    b: BlsScalar,
    c: BlsScalar,
    d: BlsScalar,
    e: JubJubScalar,
    f: JubJubAffine,
}

impl Circuit for TestCircuit {
    const CIRCUIT_ID: [u8; 32] = [0xff; 32];
    fn gadget(
        &mut self,
        composer: &mut TurboComposer,
    ) -> Result<(), Error> {
        let a = composer.append_witness(self.a);
        let b = composer.append_witness(self.b);

        // Make first constraint a + b = c
        let constraint = Constraint::new()
            .left(1)
            .right(1)
            .public(-self.c)
            .a(a)
            .b(b);

        composer.append_gate(constraint);

        // Check that a and b are in range
        composer.component_range(a, 1 << 6);
        composer.component_range(b, 1 << 5);

        // Make second constraint a * b = d
        let constraint = Constraint::new()
            .mult(1)
            .output(1)
            .public(-self.d)
            .a(a)
            .b(b);

        composer.append_gate(constraint);

        let e = composer.append_witness(self.e);
        let scalar_mul_result = composer
            .component_mul_generator(e, dusk_jubjub::GENERATOR_EXTENDED);
        // Apply the constrain
        composer.assert_equal_public_point(scalar_mul_result, self.f);
        Ok(())
    }

    fn public_inputs(&self) -> Vec<PublicInputValue> {
        vec![self.c.into(), self.d.into(), self.f.into()]
    }

    fn padded_gates(&self) -> usize {
        1 << 11
    }
}

// Now let's use the Circuit we've just implemented!

let rng = XorShiftRng::from_seed([
    0x59, 0x62, 0xbe, 0x5d, 0x76, 0x3d, 0x31, 0x8d, 0x17, 0xdb, 0x37,
    0x32, 0x54, 0x06, 0xbc, 0xe5,
]);
let pp = PublicParameters::setup(1 << 12, rng).unwrap();
// Initialize the circuit
let mut circuit = TestCircuit::default();
// Compile the circuit
let (pk, vd) = circuit.compile(&pp).unwrap();
// Prover POV
let proof = {
    let mut circuit = TestCircuit {
        a: BlsScalar::from(20u64),
        b: BlsScalar::from(5u64),
        c: BlsScalar::from(25u64),
        d: BlsScalar::from(100u64),
        e: JubJubScalar::from(2u64),
        f: JubJubAffine::from(
            dusk_jubjub::GENERATOR_EXTENDED * JubJubScalar::from(2u64),
        ),
    };
    circuit.prove(&pp, &pk, b"Test").unwrap()
};
// Verifier POV
let public_inputs: Vec<PublicInputValue> = vec![
    BlsScalar::from(25u64).into(),
    BlsScalar::from(100u64).into(),
    JubJubAffine::from(
        dusk_jubjub::GENERATOR_EXTENDED * JubJubScalar::from(2u64),
    )
    .into(),
];
TestCircuit::verify(
    &pp,
    &vd,
    &proof,
    &public_inputs,
    b"Test",
).unwrap();
```

### Features

This crate includes a variety of features which will briefly be explained below:
- `alloc`: Enables the usage of an allocator and with it the capability of performing `Proof` constructions and 
  verifications. Without this feature it **IS NOT** possible to prove or verify anything. 
  Its absence only makes `dusk-plonk` export certain fixed-size data structures such as `Proof` which can be useful in no_std envoirments where we don't have allocators either.
- `std`: Enables `std` usage as well as `rayon` parallelisation in some proving and verifying ops. 
  It also uses the `std` versions of the elliptic curve deps, which utilises the `parallel` feature 
  from `dusk-bls12-381`. By default, this is the feature that comes enabled with the crate.
- `trace`: Enables the Circuit debugger tooling. This is essentially the capability of using the 
  `StandardComposer::check_circuit_satisfied` function. The function will output information about each circuit gate until 
  one of the gates does not satisfy the equation, or there are no more gates. If there is an unsatisfied gate 
  equation, the function will panic and return the gate number.
- `trace-print`: Goes a step further than `trace` and prints each `gate` component data, giving a clear overview of all the 
  values which make up the circuit that we're constructing. 
  __The recommended method is to derive the std output, and the std error, and then place them in text file 
    which can be used to efficiently analyse the gates.__
- `canon`: Enables `canonical` serialisation for particular data structures, which is very useful in integrating
  this library within the rest of the Dusk stack - especially for storage purposes.


## Documentation

There are two main types of documentation in this repository:

- **Crate documentation**. This provides info about all of the functions that the library provides, as well
  as the documentation regarding the data structures that it exports. To check this, please feel free to go to
  the [documentation page](https://docs.rs/dusk-plonk/) or run `make doc` or `make doc-internal`.

- **Notes**. This is a specific subset of documentation which explains the key mathematical concepts
  of PLONK and how they work with mathematical demonstrations. To check it, run `make doc` and open the resulting docs,
  which will be located under `/target` with your browser.

## Performance

Benchmarks taken on `Intel(R) Core(TM) i9-9900X CPU @ 3.50GHz`
For a circuit-size of `2^16` constraints/gates:

- Proving time: `5.46s`
- Verification time: `9.34ms`. **(This time will not vary depending on the circuit-size.)**

For more results, please run `cargo bench` to get a full report of benchmarks in respect of constraint numbers.

## Acknowledgements

- Reference implementation AztecProtocol/Barretenberg
- FFT Module and KZG10 Module were taken and modified from zexe/zcash and scipr-lab, respectively.

## Licensing

This code is licensed under Mozilla Public License Version 2.0 (MPL-2.0). Please see [LICENSE](https://github.com/dusk-network/plonk/blob/master/LICENSE) for further info.

## About

Implementation designed by the [dusk](https://dusk.network) team.

## Contributing

- If you want to contribute to this repository/project please, check [CONTRIBUTING.md](https://github.com/dusk-network/plonk/blob/master/CONTRIBUTING.md)
- If you want to report a bug or request a new feature addition, please open an issue on this repository.
