---
title: Package a signed JAR
description: That's your CorDapp in a file
slug: package-a-signed-jar
aliases: [
  "/operations/package-jar/"
]
menu:
  main:
    parent: corda-advanced-concepts
    weight: 140
weight: 140
---

In this section, you will deploy your CorDapp on the node you create.

For deployment, your CorDapp will be packaged as a JAR, really a fancy ZIP file. What your CorDapp needs to include is anything that is necessary for it to work, minus what is already part of the Corda platform, i.e. the _Corda JARs_. That is why you saw [dependencies](https://docs.corda.net/docs/corda-os/4.5/cordapp-build-systems.html#corda-dependenciess) like [`cordaCompile` and `cordaRuntime`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/build.gradle#L94-L96) in `build.gradle`.

It also needs to exclude from packaging the other CorDapps it uses and that will be installed separately, which is why you saw the [`cordapp` dependency](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/build.gradle#L103-L107). Notice too how your project modules are declared as [`cordapp`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/build.gradle#L99-L100) so that they can be packaged independently from each other and from future hypothetical modules.

## How to Build a CorDapp

To build a CorDapp, simply execute this command inside your terminal under your project's path:

{{<MultiCodeBlock>}}

```shell
$ ./gradlew :module-name:jar
```

```powershell
C:\020-first-token> gradlew :module-name:jar
```

{{</MultiCodeBlock>}}

Where you replace `module-name` with your module's name, for instance:

{{<MultiCodeBlock>}}

```shell
$ ./gradlew :contracts:jar
```

```powershell
C:\020-first-token> gradlew :contracts:jar
```

{{</MultiCodeBlock>}}

Of course, you can also use the larger `build` task, but keep in mind that it will also run the tests. So choose the task that serves you.

The built JAR file(s) can be found in their respective `build/libs` folder.

{{<ExpansionPanel title="Troubleshooting">}}

Do check your Java version if you are facing any issue.

Keep in mind that an error like this:

```
> Task :workflows:compileJava FAILED
030-tokens-sdk/workflows/src/main/java/com/template/flows/IssueFlows.java:11: error: package javafx.util does not exist
import javafx.util.Pair;
```
Can in fact be a Java version error.

{{</ExpansionPanel>}}

## Why Sign Contract CorDapps for Production?

When you used the `deployNodes` facility to create a development network, and created a state, your CorDapp's state recorded the hash of the _contracts_ JAR file that created it. This constraint ensures reproducibility now and in the future, at the cost of flexibility. What if you wanted to make changes to what your contract verifies? It is possible to upgrade a given state, as you will learn in a later chapter, but at the cost of a few contorsions.

Now, if you reasonably expect that your contracts CorDapp will need some upgrade in the future, to facilitate the transition, you would be well-advised to choose to sign it and enforce a [signature constraint](https://docs.corda.net/docs/corda-os/4.5/api-contract-constraints.html#reasons-for-contract-constraints). This offers other advantages including accepting new contracts CorDapps if they have known signatories.

So, in effect, signing a contracts CorDapp jar file is a quasi-requirement of any Corda node running in production mode. If you wanted your dev nodes to run in production mode, you could have used [`devMode false`](https://docs.corda.net/docs/corda-os/4.5/generating-a-node.html#optional-configuration) inside `node.conf`. The node will not load any contracts CorDapp that is not signed, or that is signed by a Corda development certificate which is done by default. If you do, you’ll see something like this inside your log file:

```groovy
[WARN] 2020-02-26T05:07:45,335Z [main] cordapp.JarScanningCordappLoader. - Not loading CorDapp [Your CorDapp] as it is signed by development key(s) only.
```

If you recall, a signed transaction is constituted of its content, its serialized bytes, which determine its id, and a signature. When a signature is added to a transaction, it is indeed _added_ to it and the signature does not change the transaction bytes or id. Similarly here, when you sign your CorDapp, you do not modify its content, instead you affix one or more signatures to it.

Note that only _contracts_ CorDapps ought to be signed as they are part of transactions, to be verified in the future, and they travel over the wire. There is no point in signing _workflows_ CorDapps as they are installed on nodes by its administrator. Of course, you separated both CorDapp types into separate modules from the start.

## How to Sign Your CorDapps

To create a signed JAR, is a roughly 2 step process:

1. A key has to be created, or reused.
2. The key has to be used to sign.

### Create a key

The key type that is used to sign a CorDapp JAR has to be stored in a file of type [PKCS12](https://en.wikipedia.org/wiki/PKCS_12), in effect a container type of file. The file itself is encrypted by a password. Inside this container, each individual key is identified by an _alias_, which you are free to choose. In this case, you will use the alias `awesome-cordapp-signer`. Each key is also encrypted by another, or same password. For simplicity, you will use the same password.

The PKCS12 can be created in 2 steps and will be eventually stored in `~/.corda-keys`. So go ahead and create the folder:

{{<MultiCodeBlock>}}

```shell
$ mkdir ~/.corda-keys
```

```powershell
C:\020-first-token> mkdir ~\.corda-keys
```

{{</MultiCodeBlock>}}

Fortunately, as part of the installation of the JDK on your computer, another executable was installed: [`keytool`](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/keytool.html).

1. Create a private key in JKS format, another container type, in the file `jar-sign-keystore.jks`.

   In the line below, complete the X500 name with your parameters, and use the same `password` value for both `storepass` and `keypass`. Don't use `password` itself, obviously. Remember too that to avoid having this sensitive line in the shell's history, prefix it with [a single space](https://stackoverflow.com/questions/6475524/how-do-i-prevent-commands-from-showing-up-in-bash-history).

    {{<MultiCodeBlock>}}

    ```shell
    $ keytool -keystore jar-sign-keystore.jks -keyalg RSA -genkey -dname "OU=, O=, L=, C=" -storepass password -keypass password -alias awesome-cordapp-signer -validity 365
    ```

    ```powershell
    C:\020-first-token> keytool -keystore jar-sign-keystore.jks -keyalg RSA -genkey -dname "OU=, O=, L=, C=" -storepass password -keypass password -alias awesome-cordapp-signer -validity 365
    ```

    {{</MultiCodeBlock>}}

   You should see:

    ```
    Generating 2,048 bit RSA key pair and self-signed certificate (SHA256withRSA) with a validity of 365 days
            for: OU=DevOps, O=Corda, L=London, C=UK
    ```

2. Migrate the JKS key to PKCS12 format.

   You will be prompted for 2 passwords, use the same value that you used to create the JKS key for both values:

    {{<MultiCodeBlock>}}

    ```shell
    $ keytool -importkeystore -srckeystore jar-sign-keystore.jks -destkeystore ~/.corda-keys/jar-sign-keystore.pkcs12 -deststoretype pkcs12
    ```

    ```powershell
    C:\020-first-token> keytool -importkeystore -srckeystore jar-sign-keystore.jks -destkeystore ~\.corda-keys\jar-sign-keystore.pkcs12 -deststoretype pkcs12
    ```

    {{</MultiCodeBlock>}}

Now erase the first file:

{{<MultiCodeBlock>}}

```shell
$ rm jar-sign-keystore.jks
```

```powershell
C:\020-first-token> del jar-sign-keystore.jks
```

{{</MultiCodeBlock>}}

Take care of your `~/.corda-keys/jar-sign-keystore.pkcs12` file as you would another key file. If you chose a folder other than `~/.corda-keys/`, make sure that you do not commit it into a Git repository.

With the key created, it is time to instruct Gradle.

### Discard signing for flows

Since one only signs contracts CorDapps, let's explicitly disable signing on the workflows CorDapp. In the workflows module's `build.gradle`:

```groovy
cordapp {
    targetPlatformVersion corda_platform_version
    minimumPlatformVersion corda_platform_version
    workflow {
        name "Template Flows"
        vendor "My Company"
        // If it's not open source, use "Proprietary"
        licence "Apache License, Version 2.0"
        versionId 1
    }
    signing {
        enabled false
    }
}
```

### A local config file

Now, to the contracts CorDapp. Gradle will need to:

* Have access to the PKCS12 file.
* Know what password to use to decrypt it.

And because neither the PKCS12 file, nor the password shall be committed into the Git repository:

* You need to create an intermediate configuration file that points to the PKCS12.
* You will pass the password on the command line.

So create a `signing.properties` file in your project's folder:

{{<MultiCodeBlock>}}

```shell
$ touch signing.properties
```

```powershell
C:\020-first-token> echo $null >> signing.properties
```

{{</MultiCodeBlock>}}

Then, inside it, add:

```properties
jar.sign.keystore=/absolute/path/to/.corda-keys/jar-sign-keystore.pkcs12
jar.sign.alias=awesome-cordapp-signer
```
Where you have replaced `/absolute/path/to/` with the correct absolute path to the key, don't use `~`. It is safe to commit this file to Git as it does not contain any private key or password. Now, it is time to direct Gradle how to sign.

### Use the key to sign

Add the below code inside the `build.gradle` file of each contracts CorDapp in your project. Note how it makes use of `getProperty()` instead of exposing the plain text password:

```groovy
cordapp {
    targetPlatformVersion corda_platform_version
    minimumPlatformVersion corda_platform_version
    contract {
        name "Template CorDapp"
        vendor "Corda Open Source"
        // If it's not open source, use "Proprietary"
        licence "Apache License, Version 2.0"
        versionId 1
    }
    signing {
        enabled true
        options {
            Properties signing = new Properties()
            file("$projectDir/../signing.properties").withInputStream { signing.load(it) }
            keystore signing.getProperty("jar.sign.keystore")
            alias signing.getProperty("jar.sign.alias")
            storepass getProperty("SIGN_PASS")
            keypass getProperty("SIGN_PASS")
            storetype "PKCS12"
        }
    }
}
```
Now that Gradle is correctly informed, it is time to package the CorDapp again. But what is this `SIGN_PASS` property? It is an argument that you pass to the command line:

{{<MultiCodeBlock>}}

```shell
$ ./gradlew -PSIGN_PASS=password :contracts:jar
```

```powershell
C:\020-first-token> gradlew -PSIGN_PASS=password :contracts:jar
```

{{</MultiCodeBlock>}}

Where you have replaced `password` by the actual password you used. That's it?

### Verify

How can you verify that it is signed indeed? Simple. Use `jarsigner`, another tool that came with the JDK:

{{<MultiCodeBlock>}}

```shell
$ jarsigner -verify -verbose -keystore ~/.corda-keys/jar-sign-keystore.pkcs12 contracts/build/libs/contracts-0.1.jar
```

```powershell
C:\020-first-token> jarsigner -verify -verbose -keystore ~\.corda-keys\jar-sign-keystore.pkcs12 contracts\build\libs\contracts-0.1.jar
```

{{</MultiCodeBlock>}}

You should obtain something like:

```
s      1110 Wed Jun 03 19:55:36 GET 2020 META-INF/MANIFEST.MF
       1080 Wed Jun 03 19:55:38 GET 2020 META-INF/AWESOME-.SF
       1186 Wed Jun 03 19:55:38 GET 2020 META-INF/AWESOME-.RSA
          0 Fri Feb 01 00:00:00 GET 1980 META-INF/
          0 Fri Feb 01 00:00:00 GET 1980 com/
          0 Fri Feb 01 00:00:00 GET 1980 com/template/
          0 Fri Feb 01 00:00:00 GET 1980 com/template/contracts/
          0 Fri Feb 01 00:00:00 GET 1980 com/template/states/
sm      493 Fri Feb 01 00:00:00 GET 1980 com/template/contracts/TokenContract$Commands$Issue.class
sm      490 Fri Feb 01 00:00:00 GET 1980 com/template/contracts/TokenContract$Commands$Move.class
sm      496 Fri Feb 01 00:00:00 GET 1980 com/template/contracts/TokenContract$Commands$Redeem.class
sm      486 Fri Feb 01 00:00:00 GET 1980 com/template/contracts/TokenContract$Commands.class
sm     8262 Fri Feb 01 00:00:00 GET 1980 com/template/contracts/TokenContract.class
sm     2746 Fri Feb 01 00:00:00 GET 1980 com/template/states/TokenState.class
sm     2281 Fri Feb 01 00:00:00 GET 1980 com/template/states/TokenStateUtilities.class
  s = signature was verified
  m = entry is listed in manifest
  k = at least one certificate was found in keystore
- Signed by "OU=, O=, L=, C="
    Digest algorithm: SHA-256
    Signature algorithm: SHA256withRSA, 2048-bit key
jar verified.
Warning:
This jar contains entries whose certificate chain is invalid. Reason: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
This jar contains signed entries that are not signed by alias in this keystore.
This jar contains entries whose signer certificate is self-signed.
This jar contains signatures that do not include a timestamp. Without a timestamp, users may not be able to validate this jar after any of the signer certificates expire (as early as 2021-06-03).
Re-run with the -verbose and -certs options for more details.
The signer certificate will expire on 2021-06-03.
```

## Cleanup

What to do with the PKCS12 file? You ought to keep it on a build machine only, and not necessarily distribute it across all developer computers. On each developer computer, in order for the `jar` task to succeed, each developer can create new self-signed keys by following the above steps.

If you do want to distribute the PKCS12 file so you can all sign with the same key, it's not recommended to send the file as it is across the internet. Instead, you can encrypt it with the recipient's public key then send it to them. A detailed article about using gpg to encrypt/decrypt files can be found [here](https://www.gnupg.org/gph/en/manual/x110.html).

## Conclusion

You have created a neatly signed CorDapp JAR.