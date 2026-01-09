---
hasThumbnail: false

---

## Overview

This was a final project for EE 123, and an end-to-end DSP challenge: Send a video throught a channel that is so bandwidth-limited that we only get about 60 APRS packets total (~10KB of payload) and still reconstruct a video with the same dimensions on the receiver (minimum PSNR or 25dB).

We used **animated PNG (APNG)** as the video container (instead of GIF) because APNG supports full-color frames without the 256-color palette limitation. 

I explored main compression algorithms, from classic lossless compressors like **LZW** and **DEFLATE**, **JPEG 2000**, and other machine learning approaches. The main compression algorithm that I ended up on was **SPIHT**, a wavelet-based embedded codec similar in spirit to **JPEG 2000**.


## End-to-End System

1. **Sender (Raspberry Pi):**
   - Load the APNG into frames.
   - Compress the frames into a compact bytestream (our best results used **SPIHT**).
   - Packetize the bytestream into **APRS/AX.25 packets**.
   - Transmit packets using the radio interface (TNC behavior via **Direwolf**).

2. **Receiver (Server-side reconstruction):**
   - Collect packets and reassemble the payload.
   - Read metadata (dimensions, frame count, etc.) and reconstruct the APNG frames.
   - Decompress the bytestream into pixel data.
   - Output a video with the **exact same frame dimensions** as the original (quality may be lower).

Note: the server was set up to **pull our decoding code from GitHub**. One header packet included our repository name, and the server cloned it and ran a specific reconstruction entrypoint to decode our submission.

## Challenges

In normal video systems, we might have megabits per second. Here we had roughly **~10KB** of payload to deliver the entire video. This makes it neccesary to decide what to preserve: spatial detail, motion smoothness, color, etc. We also need to spend bits efficiently. The headers, metadata, and redundancy all matter. Ideally the algorithm can also degrade gracefully so that if we can only afford part of the bitstrea, the receiver should still produce something recognizable.

This is one reason I favored **embedded** codecs like SPIHT: they naturally produce a **progressive bitstream** where earlier bits give coarse quality and later bits refine it.


## Compression Approaches

Here are the main families of approaches I tried:

### 1) LZW (dictionary compression)

LZW (Lempel–Ziv–Welch) is a **lossless** compressor that builds a dictionary of repeated sequences. It’s great when the data has recurring patterns. 

**How it works:**
- Scan symbols left-to-right.
- Maintain a dictionary of sequences you’ve seen.
- Replace repeated sequences with shorter dictionary codes.

It is a bit limited here because raw video frames (especially full-color) often look “random” to a dictionary coder unless we first transform/predict the data. Without a transform (like wavelets/DCT) or a predictor (frame differencing / motion compensation), LZW often won’t shrink video enough to meet the ~10KB constraint.

### 2) LZ77 + Huffman = DEFLATE (what ZIP/PNG use)

DEFLATE combines:
- **LZ77**: finds repeated substrings using a sliding window and encodes them as (distance, length) references
- **Huffman coding**: entropy-codes symbols so frequent ones use fewer bits

It is also a lossless algorithm and is very good for strong general-purpose compression. PNG uses DEFLATE internally, so it was a natural target when working with PNG/APNG.

**How it works:**
- LZ77 turns repetition into back-references.
- Huffman assigns shorter codes to more probable symbols.

While the lossless nature was great, we had extreme bandwidth limits, where we needed some lossy compression to preserve perceptual quality. DEFALTE can't choose to discard less important details, it must preserve everything. However, when DEFLATE does achieve high compression, PSNR can be effectively infinite because the reconstruction is identical—lossless.

### 3) JPEG (DCT-based)

JPEG transforms image blocks with the **DCT** and then quantizes coefficients (lossy). It’s extremely effective for natural images.

**The key idea:**
- DCT concentrates energy into a few low-frequency coefficients.
- Quantization throws away high-frequency components (fine detail) first.
- Entropy coding compresses what remains.

This fits the idea of a video under a byte limit. It spends bits on what matters. However, block artifacts and inefficiency at very low bitrates, especially for sharp edges and text-like content made it struggle for this project.

### 4) JPEG 2000 (wavelet-based)

JPEG 2000 replaces block-DCT with a **wavelet transform**. Instead of block artifacts, wavelets give multi-resolution structure.

**Core concept:**
- Wavelets represent the image at multiple scales:
  - **LL**: coarse approximation
  - **LH/HL/HH**: detail bands (horizontal/vertical/diagonal)
- Many wavelet coefficients are small and can be truncated/quantized.

JPEG 2000 is known for good quality at low bitrates and supports progressive transmission.

### 5) SPIHT (Set Partitioning in Hierarchical Trees)

SPIHT is a wavelet-based codec that exploits two facts:

1. **After a wavelet transform**, most coefficients are small (a few carry most energy).
2. **Coefficients are correlated across scales** (a large coefficient at coarse scale often implies large coefficients in related positions at finer scales).

SPIHT organizes coefficients into **spatial orientation trees** and encodes significance progressively.

SPIHT outputs an **embedded bitstream**:
- We can stop decoding early and still recover a lower-quality version.
- Every extra bit generally improves quality (fine refinements).

### How SPIHT works
After wavelet decomposition, SPIHT iterates over bit-planes from most significant to least:

- Maintain three sets (lists):
  - **LIP** (List of Insignificant Pixels): individual coefficients not yet significant
  - **LIS** (List of Insignificant Sets): groups/trees of coefficients not yet significant
  - **LSP** (List of Significant Pixels): coefficients already found significant

- For each threshold (bit-plane), SPIHT does:
  1. **Sorting pass**
     - Test items in LIP: is `|c| >= 2^k`?
       - If yes, output `1`, output sign, move to LSP
       - If no, output `0`
     - Test sets in LIS:
       - If a set is significant, split it (partitioning) and continue testing children/descendants

  2. **Refinement pass**
     - For coefficients already significant (in LSP), output the next refinement bit.

This sort then refine loop is why the bitstream is progressive. Early passes locate the big coefficients while later passes refine them.

#### RGB handling
A practical way to handle color is:
- apply wavelets + SPIHT per channel (R, G, B), or
- convert to YCbCr and allocate more bits to luminance (often better perceptually).

Our implementation treated color channels separately for simplicity. This can work, but it may not exploit cross-channel correlation as well as YCbCr.


### 6) Neural approach (D-NERV)

Neural representation methods learn a compact model of a video such that the network weights become the “compressed” form. This could be strong because it can exploit temporal redundancy naturally. However, it was hard in this project because we have to submit not only the data but model parameters. It also didn't fit well for our timeline and constraints.

## Transmission + Reconstruction

Other than compression, we also need:
- **Packetization**: breaking the bytestream into APRS-sized chunks
- **Metadata**: frame dimensions, frame count, decoding parameters, ordering/sequence IDs
- **Robustness**: if packets arrive out of order, the receiver must reassemble correctly
- **Server reproducibility**: the decoding script must run exactly as expected on the server

## Implementation notes
- SPIHT implementation used a Haar wavelet decomposition and encoded coefficients using significance testing and refinement passes.
- End-to-end pipeline: APNG frames → per-frame compression → packetization → APRS transmit → server reassembly → decompression → APNG reconstruction.