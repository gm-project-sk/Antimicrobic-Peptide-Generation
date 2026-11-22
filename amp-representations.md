# AMP representations

## Chemical representations for generating new AMP candidates

The project goal is to learn from known AMPs and sample new peptide identities. Here, a candidate counts as new to training when its chemical structure differs from every training peptide after consistent normalization, including stereochemistry and modifications. Similarity to its closest training example is a separate measurement.

### SMILES-derived monomer tokens in GPepT

GPepT is a direct match for the generation goal. The starting object is a peptide SMILES string. The model reads one token per recognized chemical building block, including noncanonical residues and terminal modifications. The authors mined their peptide dataset from ChEMBL and fine-tuned on 205 entries labeled active against *E. coli*.

**Formal definition.** Let `s` be the peptide SMILES, `M(s) = (b1, ..., bL)` its ordered list of building blocks returned by Monomerizer, and `id(b)` the vocabulary ID of block `b`. The peptide becomes the integer vector `t[i] = id(bi)`. At training step `i`, the input is the prefix through position `i` and the target is `t[i+1]`. During generation, the model samples one building-block ID at a time.

**Worked example.** Suppose a structure contains glycine, D-tryptophan, and serine. In illustrative notation its tokens are `[Gly, D-Trp, Ser]`. One training pair is `[Gly, D-Trp] → Ser`. Distinct tokens for D-Trp and L-Trp let the output specify which stereoisomer occurs at that position. The authors sampled 500 candidate sequences after fine-tuning.

[GPepT — Oikawa et al., primary paper and methods](https://doi.org/10.1021/acsmedchemlett.5c00375)

### Atom-pair fingerprints for generating antimicrobial analogues

The study *Exploring the Sequence Space of Antimicrobial Peptide Dendrimers* generated analogues of known branched AMPs. It used a genetic algorithm with an explicit chemical similarity score. This is an example of generating similar peptides from a reference structure. Its search is driven by the reference and a scoring rule, so there is no AMP training set for this algorithm.

**Formal definition.** Let `G = (V, E)` be a candidate's atom-and-bond graph and `F(G) = MXFP(G)` its fixed-length macromolecule extended atom-pair fingerprint. The fingerprint summarizes how pairs of atoms with selected chemical properties are spaced along bond paths. For a reference graph `R`, each candidate receives a distance `D(F(G), F(R))`. The algorithm mutates candidate peptides and selects candidates using this fingerprint distance.

**Input and output.** The input is a reference peptide structure. The algorithm outputs a collection of new peptide structures. For example, a mutation changes a building block in a branched peptide, which changes its atom graph. The algorithm recomputes its fingerprint and compares it with the reference. The fingerprint scores a proposed structure; the mutation step produces it.

[AMP application — Exploring the Sequence Space of Antimicrobial Peptide Dendrimers](https://doi.org/10.1002/ijch.202300096) · [Original PDGA algorithm](https://doi.org/10.1021/acs.jcim.9b01014)

### How SELFIES, whole-peptide SMILES, and direct graph generation fit

| Representation | Formal object used during generation | Scope of the paper |
|---|---|---|
| SELFIES tokens | Encode molecular structure `G` as `(u1, ..., uK)` in the SELFIES vocabulary. A masked diffusion model predicts missing tokens and produces a decodable molecular string. | ApexOracle uses DBAASP measurements in an activity predictor that guides molecular generation. Its generator was pretrained on a broader corpus. |
| Whole-peptide SMILES tokens | Tokenize the full SMILES string as `(u1, ..., uK)`. A token can cover part of a residue. A masked diffusion model predicts tokens and outputs a new peptide SMILES. | PepTune demonstrates therapeutic peptide generation. DBAASP-based AMP training was unverified. |
| Atom-and-bond graph | One sample is `(X, B)`, where `X[i]` gives atom attributes and `B[i,j]` gives the bond category, including no bond. A direct graph generator samples both arrays. | DiGress demonstrates molecular graph generation. A direct DBAASP-trained AMP generator using this representation was unverified in this search. |

[SELFIES — ApexOracle manuscript](https://openreview.net/pdf?id=jXVC4kaocW) · [SMILES — PepTune](https://pmc.ncbi.nlm.nih.gov/articles/PMC11703324/) · [Direct molecular graphs — DiGress](https://arxiv.org/abs/2209.14734)

## Integer and one-hot encodings

Both encodings take the same input, a peptide sequence `x` of length `L`. Let `id(a)` be the position of amino acid `a` in this fixed alphabet, counting from 1:

```text
A C D E F G H I K L M N P Q R S T V W Y
```

### Formal definitions

**Integer encoding.** The output is a vector `t` of length `L`. At each sequence position `i`, set `t[i] = id(x[i])`. Each integer from 1 to 20 serves as a residue label.

**One-hot encoding.** The output is a binary matrix `X` with `L` rows and 20 columns. Each token ID specifies which column contains the single `1` in its row:

```text
X[i, j] = 1  if j = t[i]
X[i, j] = 0  otherwise
```

### Example

For `GAS`, the integer encoding is `[6, 1, 16]`. Those three numbers tell you where to place the ones in the corresponding `3 × 20` matrix:

```text
         A C D E F G H I K L M N P Q R S T V W Y
1 (G)    0 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
2 (A)    1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
3 (S)    0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 0
```

Both forms preserve residue identity and order. For activity classification, the measured activity supplies a separate training label.

- **Integer token IDs.** BroadAMP-GPT tokenizes residues for next-amino-acid prediction. [BroadAMP-GPT paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC12203862/)
- **One-hot encoding.** AMPlify uses one 20-dimensional binary vector per residue. Its positive training examples came from APD3 and DADP. [AMPlify paper](https://doi.org/10.1186/s12864-022-08310-4)

### AMP generators using residue tokens

- **Capecchi, Cai et al.** The recurrent generator learns to predict the next residue from the preceding tokens. Its training sequences came from peptides selected using DBAASP activity measurements. [Paper](https://doi.org/10.1039/D1SC01713F)
- **PepGAN.** The generator produces a sequence of residue tokens. An activity predictor scores proposed sequences and supplies feedback during training. DBAASP was one of four sources of peptide sequences. [Paper](https://doi.org/10.1021/acsomega.0c02088)
- **ARCADIAMP.** The initial generator trained on 13,950 DBAASP sequences. During training it learns to recover residue identities from corrupted tokens. During generation it repeatedly updates a noisy token sequence to produce a candidate peptide. [Paper](https://www.nature.com/articles/s41467-026-75030-8)

## Learned residue embeddings

The input is the token ID vector `t` of length `L`. Let `E` be a shared table of real numbers with 20 rows and `d` columns. Each row corresponds to one amino acid, and `d` is the chosen embedding size.

The output is a matrix `H` with `L` rows and `d` columns. At sequence position `i`, copy row `t[i]` from `E`.

```text
H[i, j] = E[t[i], j]  for i = 1, …, L and j = 1, …, d
```

For `GAS`, the IDs `[6, 1, 16]` select rows 6, 1, and 16 in that order. With `d = 2` and illustrative values, the next model layer receives:

```text
H = [[ 0.2,  0.8],    G
     [ 0.7, -0.1],    A
     [-0.3,  0.5]]    S
```

During training, the optimizer adjusts the values in `E` to reduce the model's loss. The token IDs stay fixed. Each occurrence of an amino acid uses the same row at this embedding layer.

If `X` is the one-hot matrix, the same lookup is `H = X × E` using matrix multiplication. The single `1` in each row of `X` selects the corresponding row of `E`.

HydrAMP's activity classifier uses a learned 128-dimensional embedding per residue. [HydrAMP paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC10017685/)

## Contextual residue embeddings

The input is the token sequence `t` of length `L`. A protein language model processes the whole sequence and returns a matrix `H = Encoder(t)` of size `L × d`, with one row of `d` numbers per residue. For example, the first row for `GAS` and the first row for `GKS` can differ even though both represent `G`. The surrounding residues influence each row.

**AMP-Diffusion** uses ESM-2 to encode each training peptide into this matrix. During training, the generator learns to denoise the continuous embedding values. During generation, it starts with noise, produces a new embedding matrix, and decodes it into an amino acid sequence. The authors report training on 19,670 peptides pooled from DBAASP, DRAMP, and APD3. [Paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC11838489/)

## Whole-peptide latent vector

The input is the token ID vector `t` for a peptide of length `L`. An encoder maps the sequence to a latent representation `z` with `k` real-valued entries. The chosen size `k` is the same for every peptide length. A decoder takes `z` and predicts a sequence of residue tokens.

```text
Encoder:  [6, 1, 16] → z = [0.4, -1.2]
Decoder:  [0.4, -1.2] → [6, 1, 16] → GAS
```

The numbers illustrate a successful reconstruction with `k = 2`. In a variational autoencoder, the encoder predicts a distribution over latent vectors. During training, the decoder receives a vector sampled from that distribution and learns to reconstruct the input sequence. After training, the decoder can also take a vector sampled from the model's prior and produce a candidate sequence.

PepVAE uses a 50-dimensional latent space and decodes sampled latent vectors into amino acid sequences. [PepVAE paper](https://doi.org/10.3389/fmicb.2021.725727)

## Physicochemical descriptors

The input is an amino acid sequence `x` of length `L`. Choose `k` property functions in advance. The output is a fixed-length vector with one value per property.

```text
D(x) = [f₁(x), …, fₖ(x)]
```

For example, take sequence length and mean hydrophobicity. Let `h(a)` be the hydrophobicity of residue `a` on a fixed scale.

```text
f₁(x) = L
f₂(x) = (h(x[1]) + … + h(x[L])) / L
```

Using the illustrative values `h(G) = 0`, `h(A) = 2`, and `h(S) = −1`, the model receives `D(GAS) = [3, 1/3]`. These two descriptors give the same vector for `SGA`, since both depend only on residue counts. Other descriptors can capture aspects of residue order.

Vishnepolsky and colleagues used calculated peptide properties in strain-specific activity classifiers. [Paper](https://doi.org/10.1093/bib/bbac233)

## Sequence similarity scores

The input is a peptide sequence `x` and a fixed, ordered list of reference sequences `r[1], …, r[k]`. The output is a vector with one comparison per reference. Using edit distance, define:

```text
v(x)[j] = edit_distance(x, r[j])  for j = 1, …, k
```

Edit distance is the minimum number of single-residue insertions, deletions, or substitutions needed to turn one sequence into the other.

For `x = GAS` and references `[GAS, GAT, AAA]`, the vector is `[0, 1, 2]`. The comparisons require zero edits, one substitution, and two substitutions, respectively.

KreinAMP uses pairwise edit distances or local alignment scores in a support vector machine. The vector above illustrates the comparisons for one peptide. [KreinAMP paper](https://doi.org/10.1039/D3DD00004D)




## Chemical graphs

The input is a peptide's chemical structure, for example an isomeric SMILES string. For standard linear peptides, the structure can be constructed from the residue sequence and standard templates once the terminal chemistry is specified. Modified residues, crosslinks, and ring closures require their corresponding chemical annotations.

### Formal definition

For the SMILES-to-graph conversion:

- **Input.** One valid SMILES string, such as `CC(=O)N`.
- **Output.** One molecular graph `G`, stored as an atom feature matrix, a list of bonded atom pairs, and a bond feature matrix. The output for `CC(=O)N` has four atom rows and three bond rows with implicit hydrogens.

```text
G = (V, E, X_atom, X_bond)
V = {1, …, N}                       one node per represented atom
E = {(i, j) : atoms i and j are bonded, with i < j}
X_atom has shape N × d_atom         one feature row per atom
X_bond has shape M × d_bond         one feature row per bond
M = number of bonds in E
```

The model can receive the two feature matrices and the list of bonded atom pairs. Atom features can include element, formal charge, and stereochemical information. Bond features can include single, double, triple, or aromatic bond type and stereochemistry where applicable. Hydrogen atoms may be implicit. The atom indices identify which rows are connected.

For example, `CC(=O)N` describes acetamide and illustrates the amide connectivity found in peptide bonds. Number the atoms in reading order:

```text
Atoms   1 = C, 2 = C, 3 = O, 4 = N
Bonds   (1, 2) single, (2, 3) double, (2, 4) single
```

A SMILES parser reads one graph from a valid string, up to atom numbering. Several strings can describe the same graph, such as `CC(=O)N` and `NC(=O)C`. Unspecified stereochemistry stays unspecified, and connectivity alone leaves 3D conformation open. To count distinct generated structures, compare canonical isomeric SMILES produced with the same toolkit and settings.




### What the models receive and produce

- **AmpHGT, classifier.** The pipeline uses RDKit to process the peptide structure into atom and bond features, adds fragment and residue nodes, and incorporates embeddings from the residue sequence. The classifier receives the resulting graph representation and predicts antimicrobial activity. [AmpHGT paper](https://doi.org/10.1186/s12915-025-02253-4)
- **PepTune, generator.** Training examples are tokenized peptide SMILES strings. The model receives a partially masked token sequence and learns to recover the original tokens. At generation time it produces a new SMILES string, which can be parsed into an atom-and-bond graph if valid. The graph is encoded as a string during generation. [PepTune paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC11703324/)
- **ApexOracle, generator and activity predictor.** This manuscript uses SELFIES, another string encoding of molecular connectivity. The generative component works with SELFIES tokens and outputs a molecular string that can be decoded to a chemical graph. DBAASP supplies peptide activity data for the prediction component. [ApexOracle manuscript](https://openreview.net/pdf?id=jXVC4kaocW)

### Direct molecular graph generation

A DBAASP training example supplies a peptide SMILES and a recorded activity measurement. Use the measurement to select the training examples, then parse each SMILES into atom types and bond types. For the selected examples, the generator learns from the molecular graphs.

```text
Training input:  a graph with some atom and bond types corrupted
Training target: the original atom and bond types
Generation input: a random graph state for a sampled atom count
Generation output: atom types and bonds defining a candidate molecule
```

DiGress demonstrates this approach on molecular datasets. Applying it to DBAASP is an adaptation. A peptide implementation needs to preserve stereochemistry and check whether each generated graph describes a valid peptide. Activity remains a prediction until measured. [DiGress paper](https://arxiv.org/abs/2209.14734)

## Chemical building blocks derived from SMILES

**GPepT** starts with peptide SMILES. Its Monomerizer identifies the chemical building blocks and converts each peptide into a sequence of building-block IDs. A transformer learns to predict the next ID, so its output can specify noncanonical residues as well as standard ones. The authors built their vocabulary using ChEMBL, fine-tuned on 205 examples labeled active against *E. coli*, and sampled 500 candidate sequences. This is a published example of using chemical structure during preprocessing and generating antimicrobial candidates one building block at a time. [GPepT paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC12359002/)

**PepINVENT** represents a peptide as a string of residue-level SMILES fragments. It masks selected residues and trains an encoder–decoder to generate the chemical strings for the missing positions. Filling the positions produces a new peptide analogue, potentially containing noncanonical residues. The paper studies general peptide generation. [PepINVENT paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC12002334/)
