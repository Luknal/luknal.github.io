---
layout: post
title: Cipher
description: A classical cipher that is able to encrypt a plaintext into a ciphertext, and decrypt a ciphertext into a plaintext. Also capable of encrypting and decrypting entire files
skills: 
  - Java
  - Object-oriented design
  - Inheritance & polymorphism

main-image: /cipher.svg
---

## Overview

This project implements a family of **classical ciphers** in Java. It can encrypt a
plaintext into a ciphertext, decrypt it back, and run the same transformation over an
entire text file. The design leans on **object-oriented principles** — an abstract base
class defines the shared contract, and each concrete cipher provides its own encryption
scheme. A `MultiCipher` then composes several ciphers into a single, layered cipher.

**Class structure:**

| Class | Role |
|-------|------|
| `Cipher` | Abstract base — defines the encrypt/decrypt contract and file handling |
| `Substitution` | General substitution cipher driven by an encoding string |
| `CaesarShift` | Substitution built by rotating the alphabet by *n* positions |
| `CaesarKey` | Substitution built from a keyword followed by the remaining letters |
| `MultiCipher` | Chains multiple ciphers together, applied in sequence |
| `Client` | Interactive console program to drive the ciphers |

---

## `Cipher` — the abstract base class

`Cipher` defines the shared interface every cipher must implement (`encrypt` / `decrypt`)
and provides the file-processing logic so subclasses don't have to repeat it. This is
where **abstraction** does the heavy lifting: file handling is written once, while the
actual transformation is deferred to subclasses.

```java
import java.util.*;
import java.io.*;

// Represents a classical cipher that is able to encrypt a plaintext into a ciphertext, and
// decrypt a ciphertext into a plaintext. Also capable of encrypting and decrypting entire files
public abstract class Cipher {
    // The valid characters allowed to be encoded or decoded by our Cipher.
    public static final String VALID_CHARS
        = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

    // Applies this Cipher's encryption scheme to the file with the given 'fileName',
    // creating a new file to store the results.
    public void encryptFile(String fileName) throws FileNotFoundException {
        fileHelper(fileName, true, "-encrypted");
    }

    // Applies the inverse of this Cipher's encryption scheme to the file with the
    // given 'fileName', creating a new file to store the results.
    public void decryptFile(String fileName) throws FileNotFoundException {
        fileHelper(fileName, false, "-decrypted");
    }

    // Reads from an input file, either encrypting or decrypting each line depending on
    // 'encrypt', printing the results to a new file with 'suffix' appended to its name.
    private void fileHelper(String fileName, boolean encrypt, String suffix)
                            throws FileNotFoundException {
        Scanner sc = new Scanner(new File(fileName));
        String out = fileName.split("\\.txt")[0] + suffix + ".txt";
        PrintStream ps = new PrintStream(out);
        while (sc.hasNextLine()) {
            String line = sc.nextLine();
            ps.println(encrypt ? encrypt(line) : decrypt(line));
        }
    }

    // Returns true if 'character' is one of the valid characters.
    public static boolean isCharValid(char character) {
        return VALID_CHARS.indexOf(character) != -1;
    }

    // Applies this Cipher's encryption scheme to 'input', returning the result.
    public abstract String encrypt(String input);

    // Applies the inverse of this Cipher's encryption scheme to 'input'.
    public abstract String decrypt(String input);
}
```

---

## `Substitution` — the general substitution cipher

Every other cipher extends `Substitution`. It maps each valid character to another based on
a validated `encoding` string, throwing exceptions for null, wrong-length, duplicate, or
invalid-character encodings so the cipher is always in a well-defined state.

```java
import java.util.*;

public class Substitution extends Cipher {

    private String encoding;

    public Substitution() {
        this.encoding = null;
    }

    public Substitution(String encoding) {
        setEncoding(encoding);
    }

    public void setEncoding(String encoding) {
        if (encoding == null) {
            throw new IllegalArgumentException();
        }
        if (encoding.length() != Cipher.VALID_CHARS.length()) {
            throw new IllegalArgumentException();
        }

        Set<Character> copy = new HashSet<>();
        for (int i = 0; i < encoding.length(); i++) {
            char current = encoding.charAt(i);
            if (!Cipher.isCharValid(current)) {
                throw new IllegalArgumentException();
            }
            if (copy.contains(current)) {
                throw new IllegalArgumentException();
            }
            copy.add(current);
        }

        this.encoding = encoding;
    }

    public String encrypt(String input) {
        if (input == null) {
            throw new IllegalArgumentException();
        }
        if (this.encoding == null) {
            throw new IllegalStateException();
        }
        String result = "";
        for (int i = 0; i < input.length(); i++) {
            char current = input.charAt(i);
            int index = Cipher.VALID_CHARS.indexOf(current);
            result += encoding.charAt(index);
        }
        return result;
    }

    public String decrypt(String input) {
        if (input == null) {
            throw new IllegalArgumentException();
        }
        if (this.encoding == null) {
            throw new IllegalStateException();
        }
        String result = "";
        for (int i = 0; i < input.length(); i++) {
            char current = input.charAt(i);
            int index = encoding.indexOf(current);
            result += Cipher.VALID_CHARS.charAt(index);
        }
        return result;
    }
}
```

---

## `CaesarShift` & `CaesarKey` — two ways to build an encoding

Both are thin subclasses of `Substitution`: they simply construct the encoding string in
different ways, then hand it to the parent. This is **inheritance reducing duplication** —
neither has to reimplement `encrypt` / `decrypt`.

`CaesarShift` rotates the alphabet by a fixed number of positions:

```java
import java.util.*;

public class CaesarShift extends Substitution {
    public CaesarShift(int shift) {
        if (shift < 0) {
            throw new IllegalArgumentException();
        }
        String encoding = ShiftEncoding(shift);
        setEncoding(encoding);
    }

    public String ShiftEncoding(int shift) {
        String original = Cipher.VALID_CHARS;
        int trueShift = shift % original.length();
        return original.substring(trueShift) + original.substring(0, trueShift);
    }
}
```

`CaesarKey` builds the encoding from a keyword, followed by every remaining unused letter:

```java
import java.util.*;

public class CaesarKey extends Substitution {
    public CaesarKey(String key) {
        if (key == null) {
            throw new IllegalArgumentException();
        }

        Set<Character> copy = new HashSet<>();
        for (int i = 0; i < key.length(); i++) {
            char current = key.charAt(i);
            if (!Cipher.isCharValid(current)) {
                throw new IllegalArgumentException();
            }
            if (copy.contains(current)) {
                throw new IllegalArgumentException();
            }
            copy.add(current);
        }
        String encoding = NewEncoding(key);
        setEncoding(encoding);
    }

    public String NewEncoding(String key) {
        String result = key;
        for (int i = 0; i < Cipher.VALID_CHARS.length(); i++) {
            char current = Cipher.VALID_CHARS.charAt(i);
            if (result.indexOf(current) == -1) {
                result += current;
            }
        }
        return result;
    }
}
```

---

## `MultiCipher` — layering ciphers together

`MultiCipher` holds a list of ciphers and applies them in order to encrypt. Decryption
walks the list **in reverse**, undoing each layer — a nice demonstration of composition and
why the order of operations matters.

```java
import java.util.*;

public class MultiCipher extends Cipher {
    private List<Cipher> ciphers;

    public MultiCipher(List<Cipher> ciphers) {
        if (ciphers == null) {
            throw new IllegalArgumentException();
        }
        this.ciphers = new ArrayList<>(ciphers);
    }

    public String encrypt(String input) {
        if (input == null) {
            throw new IllegalArgumentException();
        }
        String result = input;
        for (Cipher current : this.ciphers) {
            result = current.encrypt(result);
        }
        return result;
    }

    public String decrypt(String input) {
        if (input == null) {
            throw new IllegalArgumentException();
        }
        String result = input;
        for (int i = ciphers.size() - 1; i >= 0; i--) {
            result = ciphers.get(i).decrypt(result);
        }
        return result;
    }
}
```

---
