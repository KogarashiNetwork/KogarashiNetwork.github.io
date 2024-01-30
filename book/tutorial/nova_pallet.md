# Nova Tutorial

In this tutorial, we are going to use [`pallet-nova`](https://github.com/KogarashiNetwork/Kogarashi/tree/master/pallet/nova) functionality by coupling with your pallet. We use [sum-storage](https://github.com/JoshOrndorff/recipes/tree/master/pallets/sum-storage) pallet as example and call [pallet-nova](https://github.com/KogarashiNetwork/Kogarashi/tree/master/pallet/nova) zero knowledge `verify` method before set storage value. If the `verify` is successful, the value can set as storage value. You can find full example [here](https://github.com/KogarashiNetwork/Kogarashi/tree/master/sample).

The steps are following.

1. Define the pallet-nova in depencencies
2. Couple the pallet-nova to your own pallet
3. Use the pallet-nova methods in your pallet
4. Define the function for IVC verification
5. Import the coupling pallet to TestRuntime
6. Test whether the function work correctly

## 1. Define the pallet-nova in depencencies
First of all, you need to define the `pallet-nova` when you start to implement your pallet. Please define as following.

- [<your-pallet>/Cargo.toml](https://github.com/KogarashiNetwork/Kogarashi/blob/master/sample/Cargo.toml#L19)
```toml
[dependencies]
pallet-nova = { git = "https://github.com/KogarashiNetwork/Kogarashi", branch = "master", default-features = false }
rand_core = {version="0.6", default-features = false }
```

The `plonk-nova` depends on `rand_core` so please import it.

## 2. Couple the pallet-nova to your own pallet

The next, the `pallet-nova` need to be coupled with your pallet. Please couple the pallet `Config` as following.

- [<your-pallet>/src/lib.rs](https://github.com/KogarashiNetwork/Kogarashi/blob/master/sample/src/lib.rs#L22)
```rs
#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;
    use pallet_nova::{PublicParams, RecursiveProof};

    /// Coupling configuration trait with pallet_nova.
    #[pallet::config]
    pub trait Config: frame_system::Config + pallet_nova::Config {
        /// The overarching event type.
        type Event: From<Event<Self>> + IsType<<Self as frame_system::Config>::Event>;
    }
```
With this step, you can use the `pallet-nova` in your pallet through `Module`.

## 3. Use the pallet-nova methods on your pallet
Next, let's use the `pallet-nova` method in your pallet. We are going to use the `verify` method which verifies the proof. We use [sum-storage](https://github.com/JoshOrndorff/recipes/tree/master/pallets/sum-storage) pallet as example and call the `verify` method before set `Thing1` storage value on `set_thing_1`. If the `verify` is successful, the `set_thing_1` can set `Thing1` value.

- [<your-pallet>/src/lib.rs](https://github.com/KogarashiNetwork/Kogarashi/blob/master/sample/src/lib.rs#L51)
```rust
    // The module's dispatchable functions.
    #[pallet::call]
    impl<T: Config> Pallet<T> {
        /// Sets the first simple storage value
        #[pallet::weight(10_000)]
        pub fn set_thing_1(
            origin: OriginFor<T>,
            val: u32,
            proof: RecursiveProof<T::E1, T::E2, T::FC1, T::FC2>,
            pp: PublicParams<T::E1, T::E2, T::FC1, T::FC2>,
        ) -> DispatchResultWithPostInfo {
            // Define the proof verification
            pallet_nova::Pallet::<T>::verify(origin, proof, pp)?;

            Thing1::<T>::put(val);

            Self::deposit_event(Event::ValueSet(1, val));
            Ok(().into())
        }
```
With this step, we can check whether the proof is valid before setting the `Thing1` value and only if the proof is valid, the value is set.

## 4. Define the function for IVC verification
We define the function that we would like to prove the validity and extend the Nova pallet config with it. You can replace `ExampleFunction` with your own function.
`invoke` method should be implemented for a native computation and `invoke_cs` method should be for r1cs constraints.

- [<your-pallet>/src/client.rs](https://github.com/KogarashiNetwork/Kogarashi/blob/master/sample/src/client.rs#L4)
```rust
#[derive(Debug, Clone, Default, PartialEq, Eq, Encode, Decode)]
pub struct ExampleFunction<Field: PrimeField> {
    mark: PhantomData<Field>,
}

impl<F: PrimeField> FunctionCircuit<F> for ExampleFunction<F> {
    fn invoke(z: &DenseVectors<F>) -> DenseVectors<F> {
        let next_z = z[0] * z[0] * z[0] + z[0] + F::from(5);
        DenseVectors::new(vec![next_z])
    }

    fn invoke_cs<C: CircuitDriver<Scalar = F>>(
        cs: &mut R1cs<C>,
        z_i: Vec<FieldAssignment<F>>,
    ) -> Vec<FieldAssignment<F>> {
        let five = FieldAssignment::constant(&F::from(5));
        let z_i_square = FieldAssignment::mul(cs, &z_i[0], &z_i[0]);
        let z_i_cube = FieldAssignment::mul(cs, &z_i_square, &z_i[0]);

        vec![&(&z_i_cube + &z_i[0]) + &five]
    }
}
```
With this step, we finish to setup the ivc prover and verifier.

## 5. Import the coupling pallet to TestRuntime
We already define customized IVC compatible function and can use the `pallet-nova` methods so we are going to import it to `TestRumtime`.

In order to use `pallet-nova` in `TestRuntime`, we need to import `pallet-nova` crate and define the pallet config to `construct_runtime` as following.

- [<your-pallet>/src/tests.rs](https://github.com/KogarashiNetwork/Kogarashi/blob/master/sample/src/tests.rs)
```rust
use crate::{self as sum_storage, client::ExampleFunction, Config};

use frame_support::{assert_ok, construct_runtime, parameter_types};

// Import `pallet_nova` and dependencies
pub use pallet_nova::*;
use rand_core::SeedableRng;

--- snip ---

construct_runtime!(
    pub enum TestRuntime where
        Block = Block,
        NodeBlock = Block,
        UncheckedExtrinsic = UncheckedExtrinsic,
    {
        System: frame_system::{Module, Call, Config, Storage, Event<T>},
        // Define the `pallet-nova` in `contruct_runtime`
        Nova: pallet_nova::{Module, Call, Storage},
        {YourPallet}: {your_pallet}::{Module, Call, Storage, Event<T>},
    }
);

--- snip ---

impl pallet_nova::Config for TestRuntime {
    type E1 = Bn254Driver;
    type E2 = GrumpkinDriver;
    type FC1 = ExampleFunction<Fr>;
    type FC2 = ExampleFunction<Fq>;
}
```

## 6. Test whether the function work correctly
The pallet-nova methods is available on your pallet so we are going to test them as following tests.

- [<your-pallet>/src/tests.rs](https://github.com/KogarashiNetwork/Kogarashi/blob/master/sample/src/tests.rs#L94)
```rust
/// The set `Thing1` storage with valid proof
#[test]
fn sums_thing_one_with_valid_proof() {
    let mut rng = get_rng();

    let pp = PublicParams::<
            Bn254Driver,
            GrumpkinDriver,
            ExampleFunction<Fr>,
            ExampleFunction<Fq>,
        >::setup(&mut rng);

    let z0_primary = DenseVectors::new(vec![Fr::from(0)]);
    let z0_secondary = DenseVectors::new(vec![Fq::from(0)]);
    let mut ivc =
        Ivc::<Bn254Driver, GrumpkinDriver, ExampleFunction<Fr>, ExampleFunction<Fq>>::init(
            &pp,
            z0_primary,
            z0_secondary,
        );
    (0..2).for_each(|_| {
        ivc.prove_step(&pp);
    });
    let proof = ivc.prove_step(&pp);

    new_test_ext().execute_with(|| {
        assert_ok!(SumStorage::set_thing_1(Origin::signed(1), 42, proof, pp));
    });

    assert_eq!(SumStorage::get_sum(), 42);
}
```
With above tests, we can confirm that your pallet is coupling with `pallet-nova` and these methods work correctly.

The notation is aligned with [Substrate tutorial](https://docs.substrate.io/reference/how-to-guides/basics/import-a-pallet/).

You can check the `pallet-nova` example [here](https://github.com/KogarashiNetwork/Kogarashi/blob/master/pallet/nova/src/tests.rs). Happy hacking!
