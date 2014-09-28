# StreamCryptor [![Build status](https://ci.appveyor.com/api/projects/status/73fb5hecxx9xjyip)](https://ci.appveyor.com/project/bitbeans/streamcryptor) [![Build Status](https://travis-ci.org/bitbeans/StreamCryptor.svg?branch=master)](https://travis-ci.org/bitbeans/StreamCryptor)
You can use StreamCryptor to encrypt and decrypt files without size limit and the need to load every file completely into memory.
StreamCryptor uses `FileStream` to read and write files in chunks, there is also an asynchronous implementations for progress reporting available: [example](../master/examples/DemoAsync.md). 

Every file contains a `EncryptedFileHeader` some `EncryptedFileChunks` and an `EncryptedFileFooter`.
Files are encrypted into [SCCEF](https://github.com/bitbeans/StreamCryptor#sccef-file-format) (StreamCryptor Chunked Encrypted File) format.

The file serialization is realised with Google`s protobuf, it has a small overhead and offers an automatic length prefix for all file parts.
All cryptographic operations are performed via [libsodium-net](https://github.com/adamcaudill/libsodium-net) and thus [libsodium](https://github.com/jedisct1/libsodium)), see [Algorithm details](https://github.com/bitbeans/StreamCryptor#algorithm-details).

To protect the senders PublicKey from beeing tracked, you should use an ephemeral key pair for every file. If you do this it isn't possible to authenticate who encrypted the file!

## Status

> Project is currently under development!

:facepunch: Don`t use this code in a live project!

:bug: It still could contain bugs.

## Installation

There is a [NuGet package](https://www.nuget.org/packages/StreamCryptor/) available.

## This project uses the following libraries

  * [libsodium-net] - A secure cryptographic library
  * [protobuf-net] - Efficient binary serialization by Google


[libsodium-net]:https://github.com/adamcaudill/libsodium-net
[protobuf-net]:https://code.google.com/p/protobuf-net/

## Requirements

This library targets **.NET 4.5**.

## SCCEF file format

### EncryptedFileHeader
- `Version` - Used to indicate the message format. Current version is 1.
- `BaseNonce` - The 16 bytes, randomly generated nonce used to generate the chunk nonces.
- `EphemeralNonce` - The 24 byte nonce for the ephemeral secret key.
- `Key` - The encrypted 32 byte ephemeral secret key to encrypt and decrypt the chunks.
- `HeaderChecksum` - The header checksum to validate the header and prevent file manipulation.
- `Filename` - The encrypted original filename, padded to 256 bytes.
- `FilenameNonce` -  The 24 byte nonce to encrypt the filename.
- `SenderPublicKey` - The 32 byte public key of the sender to guarantee the recipient can decrypt the file.
- `UnencryptedFileLength` - The file length of the unencrypted file.

### EncryptedFileChunk
- `ChunkNumber` - Chunk number, starting at 0.
- `ChunkNonce` - Combined chunk nonce (16 byte BaseNonce from the header || 8 byte ChunkNumber)
- `ChunkLength` - The length of the chunk in bytes.
- `ChunkIsLast` - Marks the chunk as last in the file (there only can be one last chunk per file).
- `ChunkChecksum` - The checksum to validate the chunk and prevent file manipulation.
- `Chunk` - The encrypted chunk content.

### EncryptedFileFooter
- `ChunkCount` - The encrypted number of chunks in the file.
- `OverallChunkLength` - The encrypted overall length of all chunks in bytes.
- `FooterNonceLength` - The 24 byte nonce to encrypt and decrypt the ChunkCount.
- `FooterNonceCount` - The 24 byte nonce to encrypt and decrypt the OverallChunkLength.
- `FooterChecksum` - The footer checksum to validate the footer and prevent file manipulation.

## Usage

### Synchronous Methods

#### Encrypt
```csharp
public static string EncryptFileWithStream(byte[] senderPrivateKey, byte[] senderPublicKey, byte[] recipientPublicKey, string inputFile, string outputFolder = null, string fileExtension = DEFAULT_FILE_EXTENSION, bool maskFileName = false)
```

```csharp
public static string EncryptFileWithStream(KeyPair senderKeyPair, byte[] recipientPublicKey, string inputFile, string outputFolder = null, string fileExtension = DEFAULT_FILE_EXTENSION, bool maskFileName = false)
```

```csharp
//overloaded version (will use the senderKeyPair.PublicKey as recipientPublicKey)
public static string EncryptFileWithStream(KeyPair senderKeyPair, string inputFile, string outputFolder = null, string fileExtension = DEFAULT_FILE_EXTENSION, bool maskFileName = false) 
```
#### Decrypt
```csharp
public static string DecryptFileWithStream(byte[] recipientPrivateKey, string inputFile, string outputFolder, bool overWrite = false)
```

```csharp
//overloaded version (keyPair.PublicKey will be ignored)
public static string DecryptFileWithStream(KeyPair keyPair, string inputFile, string outputFolder, bool overWrite = false)
```

### Asynchronous Methods

#### Encrypt
```csharp
public static async Task<string> EncryptFileWithStreamAsync(byte[] senderPrivateKey, byte[] senderPublicKey, byte[] recipientPublicKey, string inputFile, IProgress<StreamCryptorTaskAsyncProgress> encryptionProgress = null, string outputFolder = null, string fileExtension = DEFAULT_FILE_EXTENSION, bool maskFileName = false)
```

```csharp
public static async Task<string> EncryptFileWithStream(KeyPair senderKeyPair, byte[] recipientPublicKey, string inputFile, IProgress<StreamCryptorTaskAsyncProgress> encryptionProgress = null, string outputFolder = null, string fileExtension = DEFAULT_FILE_EXTENSION, bool maskFileName = false)
```

```csharp
//overloaded version (will use the senderKeyPair.PublicKey as recipientPublicKey)
public static async Task<string> EncryptFileWithStream(KeyPair senderKeyPair, string inputFile, IProgress<StreamCryptorTaskAsyncProgress> encryptionProgress = null, string outputFolder = null, string fileExtension = DEFAULT_FILE_EXTENSION, bool maskFileName = false) 
```
#### Decrypt
```csharp
public static async Task<string> DecryptFileWithStreamAsync(byte[] recipientPrivateKey, string inputFile, string outputFolder, IProgress<StreamCryptorTaskAsyncProgress> decryptionProgress = null, bool overWrite = false)
```

```csharp
//overloaded version (keyPair.PublicKey will be ignored)
public static async Task<string> DecryptFileWithStream(KeyPair keyPair, string inputFile, string outputFolder, IProgress<StreamCryptorTaskAsyncProgress> decryptionProgress = null, bool overWrite = false)
```

The current implementation of the **Asynchronous Methods** could be run more smoothly (i`am working on this). :v:
The Problem is, neither libsodium-net nor protobuf-net has an async model.

Some example code [AsyncDemo](examples/DemoAsync.md)

### And some fixed parameters
```csharp
private const int CURRENT_VERSION = 1;
private const int MIN_VERSION = 1;
private const int CHUNK_LENGTH = 1048576; //~1MB
private const int CHUNK_COUNT_START = 0;
private const int CHUNK_MIN_NUMBER = 0;
private const int CHUNK_BASE_NONCE_LENGTH = 16;
private const int CHUNK_CHECKSUM_LENGTH = 64;
private const int HEADER_CHECKSUM_LENGTH = 64;
private const int FOOTER_CHECKSUM_LENGTH = 64;
private const int NONCE_LENGTH = 24;
private const int MAX_FILENAME_LENGTH = 256;
private const int ASYNC_KEY_LENGTH = 32;
private const int MASKED_FILENAME_LENGTH = 11;
private const string DEFAULT_FILE_EXTENSION = ".sccef"; //StreamCryptor Chunked Encrypted File
private const string TEMP_FILE_EXTENSION = ".tmp";
```

## Chunk length

I have done some time tests with different CHUNK_LENGTH`s and a **1GB** testfile, here are the results on **my** system:

|             | 524288      | 1048576     | 52428800    | 104857600   |
| :----------- | :-----------: | :-----------: | :-----------: | :-----------: |
| **Encrypt**     | ~26s        | ~26s        | ~32s        | ~32s        |
| **Decrypt**     | ~26s        | ~25s        | ~28s        |   ~28s      |

## File overhead

The produced overhead of the encrypted files:

|             | 1 KB      | 1 MB     | 100 MB    | 1000 MB   |
| :----------- | :-----------: | :-----------: | :-----------: | :-----------: |
| **Encrypted**     | +83%        | +0.1%        | +0.01%        |   +0.01%      |

## Algorithm details

|             | Using      | libsodium     | 
| :----------------------- | :-----------: | :-----------: | :-----------: |
| **Hashing (checksums)**    | Blake2b        |[documentation](http://doc.libsodium.org/hashing/generic_hashing.html) | 
| **Secret-key authenticated encryption**     | XSalsa20/Poly1305 MAC        | [documentation](http://doc.libsodium.org/secret-key_cryptography/authenticated_encryption.html)       | 
| **Public-key authenticated encryption**    | XSalsa20/Poly1305 MAC/Curve25519        | [documentation](http://doc.libsodium.org/public-key_cryptography/authenticated_encryption.html)        |

## Help wanted
See https://github.com/bitbeans/StreamCryptor/issues/3

## Why
Inspired by https://github.com/jedisct1/libsodium/issues/141 and the [nacl-stream-js](https://github.com/dchest/nacl-stream-js) project.

## License
[MIT](https://en.wikipedia.org/wiki/MIT_License)
