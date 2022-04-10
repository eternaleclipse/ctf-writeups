# PlaidCTF 2022 I_C_U Writeup

__Intro__

**I_C_U** was an exciting challenge by **f0xtr0t** and [@thebluepichu](https://twitter.com/thebluepichu) for PlaidCTF 2022. It was categorized "Misc" and included two flags, scoring for 100 and 200 points.

Reading the challenge description, the theme here is around organizing images by similarity.

Looking at the code (in Rust), some things I noticed:
- We provide two files that must pass some checks
- Some JPEG / PNG header checks
- A text recognition algorithm - Tesseract OCR
- A [Perceptual hashing](https://en.wikipedia.org/wiki/Perceptual_hashing) algorithm - [img_hash](https://github.com/abonander/img_hash)

__So, what is a Perceptual hash?__

As it turns out, it's technology that is being researched and developed in the recent years.

As opposed to cryptographic hashes (such as SHA, MD5, NTLM etc) that are supposed to be as unique as possible and prevent collision attacks, perceptual hashes are intended to produce the same, or similar results for inputs that are perceptually similar to humans. 
This means that the algorithm should give us the same hash for two "Similar" images.

Perceptual hashes are being used in reverse image searches, video copy detection, finding similar music, face matching among other interesting fields and applications. 
They can be considered a close relative of "Fuzzy hashes" (such as [ssDeep](https://ssdeep-project.github.io/)) that are heavily used in Forensics and Malware detecion / classification.

__Setup__

The provided archive conveniently contains a `Dockerfile` so I don't have to setup anything locally on my machine.

When initially building the docker image, there was an error about a missing module that is supposed to contain the flags.
Instead of creating the module, I manually patched `main.rs` with some flags:
```rust
// mod secret;
// use crate::secret::{FLAG, FLAG_TOO};
...
  println!("Congratulations: {}", "flag1");
...
  println!("Wow, impressive: {}", "flag2");

```

Building and running the image (with a host [volume](https://docs.docker.com/storage/volumes/) for easy file access from the host):

```bash
docker build -t icu .
docker run --rm -it icu -v /opt/hostvol:/vol bash
```

Running inside the container, let's test the program with the provided sample images:
```sh
root@8f173908e9c0:/icu# ./target/release/i_c_u /vol/img1.png /vol/img2.png 
Image1 hash: ERsrE6nTHhI=
Image2 hash: AaXn5FkzdkA=
Hamming Distance: 31
Image 1 text: Sudo please
Image 2 text: give me the flag
Try again
```

Cool, let's get started!

## Part I - Myopia

Let's go through the code and see what it does with our images.

We provide the program with two files, either directly or through `stdin` with strings containing base64-encoded images.
When base64 is used, the (encoded) length cannot exceed 200,000 bytes.
```rust
println!("Image 1 (base64):");
let mut s: String = String::new();
std::io::stdin().read_line(&mut s).unwrap();

if s.len() > 200_000 {
  println!("Too big");
  std::process::exit(2);
}
```

After reading the files, a simple header check is made to ensure they are either PNG or JPEG.
```rust
let suffix = {
  if b1[0..4] == b2[0..4] && b1[0..4] == [0x89, 0x50, 0x4e, 0x47] {
    "png"
  } else if b1[6..10] == b2[6..10] && b1[6..10] == [0x4a, 0x46, 0x49, 0x46] {
    "jpg"
  } else {
    println!("Unknown formats");
    std::process::exit(3);
  }
};
```

The images are being hashed with the perceptual hash.
```rust
let hash1 = hasher.hash_image(&image1);
let hash2 = hasher.hash_image(&image2);

println!("Image1 hash: {}", hash1.to_base64());
println!("Image2 hash: {}", hash2.to_base64());
```

The program calculates the [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) between the two hashes - this is just the number of bits that are different.
```rust
let dist = hash1.dist(&hash2);
println!("Hamming Distance: {}", dist);
```

Tesseract is used to extract English text from the images.
```rust
let text1 = tesseract::ocr(&fn1, "eng")
        .unwrap()
        .trim()
        .split_whitespace()
        .collect::<Vec<_>>()
        .join(" ");
    let text2 = tesseract::ocr(&fn2, "eng")
        .unwrap()
        .trim()
        .split_whitespace()
        .collect::<Vec<_>>()
        .join(" ");

    println!("Image 1 text: {}", text1);
    println!("Image 2 text: {}", text2);
```

__The checks__

There is a different set of checks for each flag. We'll focus on the first flag for now.

```rust
if dist == 0
  && text1.to_lowercase() == "sudo please"
  && text2.to_lowercase() == "give me the flag"
  && hash1.to_base64() == "ERsrE6nTHhI="
{
  println!("Congratulations: {}", "flag1");
} else if ...
```

So, to get the flag we need:
- Valid PNG or JPEG magic header at the start of the file
- The hashes of the images should be equal (dist == 0 means all bits are equal)
- The recognized text (case-insensitive) of the first file should be "sudo please"
- The recognized text (case-insensitive) of the second file should be "give me the flag"
- The hash for both images should be `ERsrE6nTHhI=`

Wait, we've seen this hash before! When we first ran the program with the samples. It's the hash for `img1.png` containing "sudo please".

Now all we have to do is create an image that has the same hash, but actually contains text that will be recognized as "give me the flag".

__Attack approaches__

Initially when I looked at this, I thought of multiple ways to solve it:
- Overflowing the `dist` integer somehow so it becomes `0` - Not an option, maximum distance would be the length of the hash string and it's really short. Besides, rust protects against int overflows
- Breaking the hamming distance function somehow - It's really simple so there should be no serious bugs, but still
- Breaking the Tesseract / `img_hash` parser - Corrupting the file format so the `img_hash` parser reads a different file than Tesseract. Maybe using another embedded image file or messing with the headers
- Bruteforce for a hash collision using [Pillow](https://github.com/python-pillow/Pillow) - Since the hash is short, it should probably be easy to find a collision. I could randomly create sets of generated images that contain "give me the flag" with different fonts, sizes, colors and positions until I get a matching hash
- Read the actual code for `hash_img` and understand what it does

__Fun with MSPaint__

Naturally, I did none of this and just opened the file with good ol' Microsoft Paint and started playing with it and seeing how the hash changes when I modify the image.

I quickly found out that:
- Small changes do not affect the hash
- Scaling the image up only slightly affects the hash, and allows more room to play with and feed the OCR with larger text
- Changing different parts of the image changes different parts of the hash
- Playing with colors can modify the value - It seemed like adding white increased the value for the specific part of the hash that was being modified

As for the OCR, it was a matter of deforming the existing ("sudo please") text so it would ignore it, and finding the correct positioning, font and size for the new text so it would only detect "give me the flag" and not add extra gibberish characters

After an hour or so, I came up with this masterpiece:

![gg](gmtf.jfif)

Note the small string at the upper-right corner - That's what's actually being read by the OCR.

## Part II - Hyperopia

Moving on from my adventures with Paint, let's look at the next flag checks:
```rust
else if dist > 0
    && text1.to_lowercase() == "give me the flag"
    && text2.to_lowercase() == "give me the flag"
    && bindiff_of_1(&fn1, &fn2)
{
    println!("Wow, impressive: {}", "flag2");
}
```

Checks for this flag:
- Header should be PNG / JPEG
- hashes should be different (by at least 1 bit)
- The text recognized for both images should be "give me the flag"
- The entire difference between the two files should be exactly 1 bit

The implementation for `bindiff_of_1` looks solid. It basically checks that length is equal, XORs the contents of the two files and checks that the sum is 1.

```rust
fn bindiff_of_1(fn1: &str, fn2: &str) -> bool {
    use std::io::Read;
    let b1: Vec<u8> = std::fs::File::open(fn1)
        .unwrap()
        .bytes()
        .collect::<Result<_, _>>()
        .unwrap();
    let b2: Vec<u8> = std::fs::File::open(fn2)
        .unwrap()
        .bytes()
        .collect::<Result<_, _>>()
        .unwrap();
    if b1.len() != b2.len() {
        return false;
    }
    b1.into_iter()
        .zip(b2.into_iter())
        .map(|(x, y)| (x ^ y).count_ones())
        .sum::<u32>()
        == 1
}
```

__Attack approaches__

Just like before, I had some thoughts on how to tackle this part:
- Overflow the sum
- Review the PNG / JPEG header and find a bit that could corrupt the file to result in an equivalent, empty, or otherwise errorneous file that produces the same hash for both images

Ain't nobody got time for that. I downscaled the image, and wrote a quick and dirty Python script to flip every bit in the image and test for success.

```python
import subprocess
import sys

def flip_bit(buf, i):
    byte_offset = i // 8
    bit_offset = i % 8
    byte = buf[byte_offset]
    byte ^= (1 << bit_offset)
    buf[byte_offset] = byte
    return buf


m1_contents = bytearray(open("/vol/m1.jpg", "rb").read())
m1_len = len(m1_contents)

for i in range(m1_len * 8):
    m2 = flip_bit(m1_contents[:], i)
    open("/vol/m2.jpg", "wb").write(m2)
    try:
        output = subprocess.check_output(["./target/release/i_c_u", "/vol/m1.jpg", "/vol/m2.jpg"])
        if b"Wow, impressive" in output:
            print(output)
            sys.exit(0)
    except subprocess.CalledProcessError:
        pass
```

During writing and testing this script I used the [Docker extension for VSCode](https://code.visualstudio.com/docs/containers/overview) which made things really easy, working directly inside the container.

After letting it run for a good 25 minutes, it finds a position where flipping the bit produces the same text and hash, and the flag appeared.

## Sending the results and getting the actual flags

Now we have to send our images to the server and get our hard-earned flags!

When I first tried base64-encoding and pasting the files in the Terminal, the server crashed with some error - `InvalidLastSymbol(4094, 78)`. Reading the [docs](https://docs.rs/base64/0.10.0/base64/enum.DecodeError.html), It seems that the last encoded base64 character (symbol) that rust received was `N`. When trying to paste it locally in my Docker container it also crashed with the same error.

This immediately made me suspect that some sort of copy-paste truncation witchery was going on. Indeed, when piping the files through `stdin`, it worked:
```bash
(base64 -w0 img1.png; echo; base64 -w0 img2.png) | ./i_c_u
```

I thought of doing the same thing with `nc`, but when interacting with the server, it has some Proof-of-Work ([hashcash](https://en.wikipedia.org/wiki/Hashcash)) command that needs to be run locally before accessing the challenge. I first tried wrapping my command with `Expect`, but eventually just wrote a Python script to do the whole thing.

```python
from socket import *
import subprocess
import base64
import sys

def read_b64(filename):
    with open(filename, 'rb') as f:
        return base64.b64encode(f.read())

s = socket(AF_INET, SOCK_STREAM)
s.connect(("icu.chal.pwni.ng", 1337))
hashcash_cmd = s.recv(1024)

if not hashcash_cmd.startwith("hashcash "):
    sys.exit(1)

output = subprocess.check_output(hashcash_cmd, shell=True)
s.send(output)

print(s.recv(1024))
s.send(read_b64("1.png") + b"\n")

print(s.recv(1024))
s.send(read_b64("2.png") + b"\n")

print(s.recv(1024))
print(s.recv(1024))
print(s.recv(1024))
print(s.recv(1024))
```

We got our flags, Awesome!

## Closing thoughts

We need more challenges like this! One aspect of this challenge I've especially enjoyed is the fact that besides finding bugs, it's about experimenting with some interesting technologies. The puzzle design aspect of it is also very good - It's an interesting problem and not some super-specific shenanigan that someone thought up about. There are multiple ways to approach it and there is no guesswork involved - just figuring out how to make two real-world algorithms work, and having two difficulty levels means you can enjoy this challenge even if you're a beginner that doesn't know how to edit binary files.
