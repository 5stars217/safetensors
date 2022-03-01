# Safetensors

This repository implements a new simple format for storing tensors
safely (as opposed to pickle) and that is still fast (zero-copy). 

##Format

8 bytes: `N`, a u64 int, containing the size of the header
N bytes: a JSON utf-8 string representing the header.
         The header is a dict like {"TENSOR_NAME": {"dtype": "float16", "shape": [1, 16, 256], "offsets": (X, Y)}}, where X and Y are the offsets in the byte buffer of the tensor data
Rest of the file: byte-buffer.


## Yet another format ?

The main rationale for this crate is to remove the need to use
`pickle` on `PyTorch` which is used by default.
There are other formats out there used by machine learning and more general
formats.


Let's take a look at alternatives and why this format is deemed interesting.
This is my very personal and probably biased view:

| Format | Safe | Zero-copy | Lazy loading | No file size limit | (B)Float-16 support | Flexibility |
| --- | --- | --- | --- | --- | --- | --- |
| pickle (PyTorch) | ✗ | ✗ | ✗ | 🗸 | 🗸  | 🗸 |
| H5 (Tensorflow) | 🗸 | ✗ | 🗸 | 🗸 | 🗸  | ~ |
| SavedModel (Tensorflow) | 🗸? | ✗? | ✗ | 🗸  | 🗸 | 🗸 | 🗸 |
| MsgPack (flax) | 🗸 | 🗸 | ✗ | 🗸 | ✗ | ✗ | ~ |
| Protobuf (ONNX) | 🗸 | ✗ | ✗ | ✗ | ✗ | ✗ | ~ |
| Cap'n'Proto | 🗸  | 🗸 | ~ | 🗸  | 🗸  | ✗ | ~ |
| SafeTensors | 🗸 | 🗸 | 🗸 | 🗸 | 🗸 | ✗ | 

Safe: Can I use a file randomly downloaded and expect not to run arbitrary code ?
Zero-copy: Does reading the file require more memory than the original file ?
Lazy loading: Can I inspect the file without loading everything ? And loading only
some tensors in it without scanning the whole file (distributed setting) ?
No file size limit: Is there a limit to the file size ?
(B)float16 support: In machine learning, float16 is becoming common as a means to reduce RAM requirements, bf16 is still a bit new but supposed to be more fit for machine learning than float16.
Flexibility: Can I save custom code in the format and be able to use it later with zero extra code ?


## Main oppositions

Pickle: Unsafe, runs arbitrary code
H5: Slow (also now discouraged for TF/Keras)
SavedModel: Tensorflow specific
MsgPack: No layout control to enable lazy loading (important for loading specific parts in distributed setting)
Protobuf: Hard 2Go max file size limit
Cap'n'proto: Float16 support is lacking. This one has the most merits, but specifying the full layout to enable real lazy loading probably requires domain knowledge which is simpler (in my view) written in bare code and enable more control over the whole thing.


## Notes

- Zero-copy: No format is really zero-copy in ML, it needs to go from disk to RAM/GPU RAM (that takes time). Also
    In PyTorch/numpy, you need a mutable buffer, and we don't really want to mutate a mmaped file, so 1 copy is really necessary to use the thing freely in user code. That being said, zero-copy is achievable in Rust if it's wanted and safety can be guaranteed by some other means.
    SafeTensors is not zero-copy for the header. The choice of JSON is pretty arbitrary, but since deserialization is <<< of the time required to load the actual tensor data and is readable I went that way, (also space is <<< to the tensor data).

- Endianness: Little-endian. This can be modified later, but it feels really unecessary atm
- Order: 'C' or row-major. This seems to have won. We can add that information later if needed.
- Stride: No striding, all tensors need to be packed before being serialized. I have yet to see a case where it seems useful to have a strided tensor stored in serialized format 

