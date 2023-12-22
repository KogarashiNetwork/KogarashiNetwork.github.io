# Ivc Tutorial

In this tutorial, we are going to import nova-ivc-pallet to substrate runtime and test its functionalities.

The steps are following.

1. Define the nova-ivc-pallet in depencencies
2. Couple the nova-ivc-pallet to your own pallet
3. Use the nova-ivc-pallet methods in your pallet
4. Import the coupling pallet to TestRuntime
5. Test whether the functions work correctly


## 1. Define the nova-ivc-pallet in depencencies
First of all, you need to define the `nova-ivc-pallet` when you start to implement your pallet. Please define as following.

- <your-pallet>/Cargo.toml
```toml
[dependencies]
nova-ivc-pallet = { git = "https://github.com/KogarashiNetwork/Kogarashi", branch = "master", default-features = false }
rand_core = {version="0.6", default-features = false }
```

The `plonk-pallet` depends on `rand_core` so please import it.

## 2. Couple the nova-ivc-pallet to your own pallet

The next, the `nova-ivc-pallet` need to be coupled with your pallet. Please couple the pallet `Config` as following.

- <your-pallet>/src/main.rs
```rs
#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;
    use nova_ivc_pallet::{PublicParams, RecursiveProof};

    /// Coupling configuration trait with nova_ivc_pallet.
    #[pallet::config]
    pub trait Config: frame_system::Config + nova_ivc_pallet::Config {}
```
With this step, you can use the `nova-ivc-pallet` in your pallet through `Module`.

## 3. Use the nova-ivc-pallet methods on your pallet
Next, let's use the `nova-ivc-pallet` method in your pallet. We are going to use the `verify` method which verifies the proof. In this tutorial, we use [sum-storage](https://github.com/JoshOrndorff/recipes/blob/master/pallets/sum-storage/src/main.rs) pallet as example and call the `verify` method before set `Thing1` storage value on `set_thing_1`. If the `verify` is successful, the `set_thing_1` can set `Thing1` value.

- <your-pallet>/src/main.rs
```rust
    // The module's dispatchable functions.
    #[pallet::call]
    impl<T: Config> Pallet<T> {
        /// Sets the first simple storage value
        #[pallet::weight(10_000)]
        pub fn set_thing_1(
            origin: OriginFor<T>,
            val: u32,
            proof: Proof,
            public_inputs: Vec<Fr>,
        ) -> DispatchResultWithPostInfo {
            // Use the proof verification
            nova_ivc_pallet::Pallet::<T>::verify(origin, proof, pp)?;

            Thing1::<T>::put(val);

            Self::deposit_event(Event::ValueSet(1, val));
            Ok(().into())
        }
```
With this step, we can check whether the proof is valid before setting the `Thing1` value and only if the proof is valid, the value is set.

## 4. Import the coupling pallet to TestRuntime and define the function for IVC verification.
We already can use the `nova-ivc-pallet` methods so we are going to import it to `TestRumtime` and define your customized IVC compatible function.

In order to use `nova-ivc-pallet` in `TestRuntime`, we need to import `nova-ivc-pallet` crate and define the pallet config to `construct_runtime` as following.

- runtime/src/main.rs
```rust
use crate::{self as sum_storage, Config};

use frame_support::dispatch::{DispatchError, DispatchErrorWithPostInfo, PostDispatchInfo};
use frame_support::{assert_ok, construct_runtime, parameter_types};

// Import `nova_ivc_pallet` and dependencies
pub use nova_ivc_pallet::*;
use rand_core::SeedableRng;

--- snip ---

construct_runtime!(
    pub enum TestRuntime where
        Block = Block,
        NodeBlock = Block,
        UncheckedExtrinsic = UncheckedExtrinsic,
    {
        System: frame_system::{Module, Call, Config, Storage, Event<T>},
        // Define the `nova-ivc-pallet` in `contruct_runtime`
        IvcPallet: nova_ivc::{Module, Call, Storage},
        {YourPallet}: {your_pallet}::{Module, Call, Storage},
    }
);
```

As the final step of runtime configuration, we define the IVC compatible function and extend the `TestRuntime` config with it. You can replace `ExampleFunction` with your own function.
Function should be implemented for a native computation and for on circuit computation as well.

- runtime/src/main.rs
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

// Should be defined on the cycle of curves.
// Function circuit is defined over the scalar of the corresponding curve.
impl nova_ivc::Config for TestRuntime {
    type E1 = Bn254Driver;
    type E2 = GrumpkinDriver;
    type FC1 = ExampleFunction<Fr>;
    type FC2 = ExampleFunction<Fq>;
}
```
With this step, we finish to setup the ivc runtime environment.

## 5. Test whether the functions work correctly
The nova-ivc-pallet methods is available on your pallet so we are going to test them as following tests.

- <your-pallet>/src/main.rs
```rust
fn main() {
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

    new_test_ext().execute_with(|| {
        for _ in 0..3 {
            let proof = ivc.prove_step(&pp);
            assert!(IvcPallet::verify(Origin::signed(1), proof, pp.clone()).is_ok());
        }
    });
}

```
With above tests, we can confirm that your pallet is coupling with `nova-ivc-pallet` and these methods work correctly. You can check the `nova-ivc-pallet` example [here](https://github.com/KogarashiNetwork/Kogarashi/blob/master/nova_pallet/src/tests.rs). Happy hacking!
