# CESS Storage Proofs PoRep

The `cess-sp-porep` is the reference implementation of _**Proof-of-Replication**_ (**PoRep**) and performs all the heavy lifting for `cess-proofs`.

_**Proof-of-Replication**_ proves that a Storage Miner is dedicating unique storage for each sector. The miners collect new client's data in a sector, run a slow encoding process (called `Seal`) and generate proof (`SealProof`) that the encoding was generated correctly.

PoRep provides two guarantees:

1. Space-hardness: Storage Miners cannot lie about the amount of space they are dedicating to CESS Network to gain more power.
2. Replication: Storage Miners are dedicating unique storage for each copy of their client's data.

The _**Proof-of-Replication**_ uses Stacked DRG (SDR) designed by [**Ben Fisch at EUROCRYPT19**](https://eprint.iacr.org/2018/702.pdf). SDR uses Depth Robust Graph to ensure the sector has been encoded with a slow and non-parallelizable sequential process.

The proof size in SDR is too large to store it in blockchain this is mostly due to the large amount of Merkle tree proofs required to achieve security. SDR verification algorithm is built using an arithmetic circuit and uses SNARKs to prove that SDR proof was evaluated correctly.

## PoRep Circuit

### Overview of entire PoRep Circuit

![1_8ngv4D_fB5WzauWAY3b2fA](https://user-images.githubusercontent.com/15224865/146317251-22961983-8b0f-49d9-8974-6c4a8de15995.png)
Credits to [Star LI](https://starli.medium.com/filecoin-porep-circuit-introduction-43415d97730c)

### StackedCircuit

StackedCircuit is the over all circuit of PoRep, defined in [proof.rs](./src/stacked/circuit/proof.rs#L28)

```rust
pub struct StackedCircuit<'a, Tree: 'static + MerkleTreeTrait, G: 'static + Hasher> {
    public_params: <StackedDrg<'a, Tree, G> as ProofScheme<'a>>::PublicParams,
    replica_id: Option<<Tree::Hasher as Hasher>::Domain>,
    comm_d: Option<G::Domain>,
    comm_r: Option<<Tree::Hasher as Hasher>::Domain>,
    comm_r_last: Option<<Tree::Hasher as Hasher>::Domain>,
    comm_c: Option<<Tree::Hasher as Hasher>::Domain>,

    // one proof per challenge
    proofs: Vec<Proof<Tree, G>>,
}
```

This includes

- public_params: StackedDrg (deep robust graph) related parameters, including the parameters of the graph itself and the number of challenges.
- replica_id: Sector copy id
- comm_d: the root of the binary tree of the original data
- comm_r: hash result of comm_r_last and comm_c
- comm_r_last: the root of the octree of the data after encoding
- comm_c: Root of the octree of column hash result
- proofs: Challenge the corresponding proof circuit

The construction of the entire circuit begins with the StackedCircuit `synthesize` interface function.

```rust
impl<'a, Tree: MerkleTreeTrait, G: Hasher> Circuit<Fr> for StackedCircuit<'a, Tree, G> {
    fn synthesize<CS: ConstraintSystem<Fr>>(self, cs: &mut CS) -> Result<(), SynthesisError> {
        let StackedCircuit {
            public_params,
            proofs,
            replica_id,
            comm_r,
            comm_d,
            comm_r_last,
            comm_c,
            ..
        } = self;
        //... // function body.
        Ok(())
    }
}

```

This function divides the circuit into two parts:

- Tree root check circuit
- Challenge node information proof circuit

The `Tree root check circuit` is fairly simple and is used for verifying comm_r is calculated currectly using comm_c and comm_r_last.
On the other hand `Challenge node proof circuit` generates challenge node proof circuit based on the size of the sector. For a `32GiB` sector `176` challenges are generated. Also called as 176 small circuits.

```rust
for (i, proof) in proofs.into_iter().enumerate() {
    proof.synthesize(
        &mut cs.namespace(|| format!("challenge_{}", i)),
        public_params.layer_challenges.layers(),
        &comm_d_num,
        &comm_c_num,
        &comm_r_last_num,
        &replica_id_bits,
    )?;
}

```

These small circuit of each challenge node is represented by `Proof` structure defined in [params.rs](./src/stacked/circuit/params.rs#L41).

### Labeling

**Labeling a Node**
The labeling function for every node in a Stacked-DRG is `Sha254` producing a 254-bit field element. A unique preimage is derived for each node-layer tuple in replicas' Stacked-DRG.

The proof circuit for Labeling calculation is to prove that a certain node is calculated correctly according to the SDR algorithm.

`generate_labels` function describes how every Stacked-DRG node is labeled for a replica. Nodes in the first layer are labeled using only DRG parents' labels, nodes in every subsequent layers are labeled using both their DRG and expander parents' labels. The first node in every layer is **not** labeled using parents.

The `LabelingProof` object can be created by calling the below function

```rust
impl<H: Hasher> LabelingProof<H> {
    pub fn new(layer_index: u32, node: u64, parents: Vec<H::Domain>) -> Self {
        LabelingProof {
            node,
            layer_index,
            parents,
            _h: PhantomData,
        }
    }
    ...
}
```

The [create_label](./src/stacked/vanilla/labeling_proof.rs#L28) function computes sha256 of `replica_id`, `layer_index` and the `node id` concatinated in a buffer and hash of the parents itself.

The following code is used to verify all the labels generated on previous step. This function just checks for quality by comparing `labeling_proof` with `label_node`

```rust
/// Verify all labels.
fn verify_labels(
    &self,
    replica_id: &<Tree::Hasher as Hasher>::Domain,
    layer_challenges: &LayerChallenges,
) -> bool {
    // Verify Labels Layer 1..layers
    ...
}
```

### Encoding

Encoding is the process by which a sector is transformed into its encoding replica. The encoding function is node-wise prime field addition, where "**node-wise**" means that every distinct slice of the sector is discretely encoded. Each distinct slice belonging to a sector is interpreted as a field element and encoded into Replica by adding its key to the slice.

The function `encode` is used to encode a sector (D) into replica (R) given an encoding key (K) derived from R's `ReplicaId`

```rust
pub fn encode<T: Domain>(key: T, value: T) -> T {
    let mut result: Fr = value.into();
    let key: Fr = key.into();

    result += key;
    result.into()
}
```

### Replication

Replication is the entire process by which a sector `D` is uniquely encoded into a replica `R`. Replication encompasses Stacked-DRG labeling, encoding `D` into `R`, and the generation of trees `TreeC` over `Labels` and `TreeR` over `R`.

A miner derives a unique `ReplicaID` for each `R` using the commitment to the replica's sector `CommD = TreeD.root` (where `TreeD` is build over the nodes of the encoded sector `D` associated with `R`).

Given a sector `D` and its commitment `CommD`, replication proceeds as follows:

Generate the `R`’s unique `ReplicaID`.
Generate `Labels` from `ReplicaID`, thus deriving the key `K` that encodes `D` into `R`.
Generate `TreeC` over the columns of `Labels` via the column commitment process.
Encode `D` into `R` using the encoding key `K`.
Generate a `TreeR: OctTree` over the replica `R`.
Commit to `R` and its associated labeling `Labels` via the commitment `CommCR`.

The function [`replicate`](./src/drg/vanilla.rs#L440) runs the entire replication process for a sector `D`.

### ReplicaID Generation

The function [generate_replica_id](./src/stacked/vanilla/params.rs#L759) describes how a miner having the `ProverID` is able to generate a `ReplicaID` for a replica `R` of sector `D`, where `D` has a unique `sectorID` and commitment `CommD`. The prover uses a unique random value _`R`_ for each `ReplicaID` generated.

### Sector Construction

A sector `D` is constructed from CESS client data where the aggregating of client data of has been preprocessed/bit-padded such that two zero bits are placed between each distinct 254-bit slice of client data. This padding process results in a sector `D` such that every 256-bit slice represents a valid 254-bit field element.

A Merkle tree `TreeD: BinTree` is constructed for sector `D` whose leaves are the 256-bit slices Di. Each `TreeD` is constructed over the preprocessed sector data `D`

### PoRep Challenges

The function [derive_internal](./src/stacked/vanilla/challenges.rs#L39) creates the PoRep challenge set for a replica `R`'s partition-`k` PoRep partition proof.

## Bellman

_"zk-SNARKs are a cryptographic technique allowing a prover to efficiently convince verifiers that the prover knows something — but without revealing the information itself. zk-SNARKs allow for secure, private interaction with unknown and untrusted parties in a blockchain setting due to their (knowledge) soundness property: a valid proof cannot be created without knowledge of the correct statment, even if they are kept private."_ by [Filecoin](https://research.protocol.ai/sites/snarks/). The main benifit of using zk-SNARKs is that it allow us to prove the validity of storage in much less space.

The `cess-proving-system` uses the "**Bellman's zk-SNARs**" implementation, mainly based on **BLS12-381** elliptic curve and realizes **Groth16**'s zero-knowledge proof system. This library is used to verify whether the zero-knowledge proof is correct and the verification process takes around several milliseconds making it relatively fast.

## License

MIT or Apache 2.0
