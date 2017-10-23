##  GPUZipCracker
*A fast GPU-based password brute-forcing tool for ZIP archives (for macOS)*

Years ago, when I wanted to store files with a reasonable amount of security/privacy, I used to use encrypted ZIP archives to store files. The problem was that ZIP archives don't actually encrypt their directory, so that metadata was store in plaintext. If you wanted to hide filenames, folder names, etc., you had to first put everything into one ZIP archive, then store that single archive into another, encrypted ZIP file.

These ZIP archives were stored using the original ZIP encryption method, often referred to as the 'traditional PKWARE encryption'. This was a fairly primitive, 96-bit encryption system that has been broken in a number of different ways, though most of these techniques seem to require at least 12-14 bytes of known plaintext.

In my Metal API studie, I was curious to see if the GPU offered a meaningful performance advantage if one wanted to simply bruteforce the password. For bruteforcing, one generally needs to know the first few plaintext characters of the file. Luckily, since the encrypted file is in itself another ZIP archive, we know the 4-byte file header.

This means that for each bruteforced iteration we need to do the following:
1. Generate a password based on the current iterator value and a characterset
2. Hash that password into a 96-bit decryption key
3. Decrypt the 11 random bytes traditional ZIP encryption uses to ensure randomness, plus the 12th byte which is used as a quick password check
4. Decrypt the first 4 total bytes of the encrypted files
5. Now we have 5 decrypted bytes we can compare against 5 known plaintext bytes.
6. If that test passes, the GPU returns the index of the password that generated it, and we do an actual full decryption to verify the password is correct (because in some cases we will run **trillions** of tests, this might actually happen several times)

> Note: The current version **only** supports decrypting ZIP archives within ZIP archives. It should be very easy to add additional file formats by setting up a table that uses the extension of the file within the ZIP archive, and having some hardcoded header bytes for each file format. How many known bytes of header do we need? Since we already have one known plaintext byte from the ZIP encryption header, I'd say the minimum is probably 2-3.
> At 2 known bytes, the GPU would trigger a positive detection roughly once every 2^24 attempts (about 16M), which would actually be pretty frequent and might dramatically reduce performance.

## Performance

The program uses a Metal-API based compute shader to bruteforce the password. It is set up to simultaneously utilize all available GPUs on a given system, but you can pick a single GPU from the command line if you wish.
Real-world performance seems to be at least 1-2 orders of magnitude faster than what you can expect with a nice Intel CPU. I've tested this on multiple Macs and performance was very encouraging. For example, on a 2013 Mac Pro, the program was able to test approximately 800 million permutations per second. The same machine was only able to achieve approximately 28 million permutations per second when running on all 8 cores of that same system.