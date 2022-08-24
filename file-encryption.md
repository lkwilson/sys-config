# Encrypting files
The standard seems to be gpg. You generate some keys, and use that for
encryption and signing.

# Generate Keys
```
gpg --gen-key
```
or
```
gpg --full-generate-key
```

# Encrypt a file
```
gpg --output file.txt.enc --recipient key-email@example.com --encrypt file.txt
shred file.txt
rm file.txt
```

# Decrypt a file
```
gpg --output file.txt --decrypt file.txt.enc
```

# Export Key
```
gpg --list-secret-keys key-email@example.com
gpg --export-secret-keys key-email@example.com
```

# Import Key
```
gpg --import-key my-key.key
gpg --edit-key key-email@example.com
gpg> trust
```

# TODO
- You can convert gpg keys to pgp text files
- See the apt guide, there's some information there since it deals with this


# Sources
- [gen key](https://www.gnupg.org/gph/en/manual/c14.html)
- [encryption](https://www.gnupg.org/gph/en/manual/x110.html)

