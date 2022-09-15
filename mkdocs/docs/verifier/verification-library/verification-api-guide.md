# Verification

After having presented an [Auth Request](./request-api-guide.md) to the user's wallet, 
it will process the request and generate a Proof that must be verified in order to authenticate the user. 
Let's see how to implement the verification.

> The proof verification follows the same flow either the Auth Request is a [Basic Auth](./request-api-guide.md#basic-auth) or a [Query-based Auth](./request-api-guide.md#query-based-auth)

=== "GoLang"

    ```go
    go get github.com/iden3/go-iden3-auth
    ```

=== "Javascript"

    ```js
    import {auth, resolver, protocol loaders, circuits} from 'js-iden3-auth';
    ```

**Unkpack the proof** 

=== "GoLang"

    ```go
    import (
        "io"
    )

    tokenBytes, err := io.ReadAll(req.Body)
    ```

=== "Javascript"

    ```js
    const getRawBody = require('raw-body')

    const raw = await getRawBody(req);
    const tokenStr = raw.toString().trim();
    ```

`req` is the post request sent by the wallet in reponse to the Auth Request posed by the Verifier. 
Unpacks the `req` 

**Initiate the verifier**

=== "GoLang"

    ```go
    verificationKeyloader := &loaders.FSKeyLoader{Dir: keyDIR}
    resolver := state.ETHResolver{
        RPCUrl:   rpcUrl,
        Contract:  contractAddress,
    }
    verifier := auth.NewVerifier(verificationKeyloader, loaders.DefaultSchemaLoader{IpfsURL: "ipfs.io"}, resolver)    
    ```

=== "Javascript"

    ```js
    const verificationKeyloader = new loaders.FSKeyLoader(keyDIR);
    const sLoader = new loaders.UniversalSchemaLoader('ipfs.io');
    const ethStateResolver = new resolver.EthStateResolver(rpcUrl, contractAddress);
    const verifier = new auth.Verifier(
        verificationKeyloader,
        sLoader, ethStateResolver,
    );
    ```

Returns an instance of a Verifier. To set up a verifier different parameters need to be passed: 

-  `keyDIR` is the path where the public verification keys for iden3 circuits are located, such as `"./keys"`. The verification key folder can be found [here](https://github.com/iden3/tutorial-examples/tree/main/verifier-integration/keys)
- `rpcUrl` is the url of your RPC node provider, such as `"https://eth-mainnet.g.alchemy.com/v2/...."`
- `contractAddress` is the address of the identity state Smart Contract. On Polygon Mainnet it is 0xb8a86e138C3fe64CbCba9731216B1a638EEc55c8.

**Execute the verification**

=== "GoLang"

    ```go
    authResponse, err := verifier.FullVerify(req.Context(), string(tokenBytes),
    authRequest.(protocol.AuthorizationRequestMessage))
    ```

=== "Javascript"

    ```js
    let authResponse: protocol.AuthorizationResponseMessage;
    authResponse = await verifier.fullVerify(tokenStr, authRequest);
    ```

Execute the verification. It verifies that the proof shared by the user satisfies the criteria set by the verifier inside the initial request.

## Verification - Under the Hood

The auth library provides a simple handler to extract all the necessary metadata from the JWZ token and execute all the verifications needed. The verification procedure that is happening behind the scenes involves: 

### Zero Knowledge Proof Verification

Starting from the circuit specific public verification key, the proof and the public inputs provided by the user it is possible to verify the proof. In this case the Proof verification involves: 

- Verification of the proof contained based on the [`Auth Circuit`](../../circuits/main-circuits.md#authentication)
- Verification of the proof contained based on the [`AtomicQuerySig Circuit`](../../circuits/main-circuits.md#credentialatomicquerysig) or [`AtomicQueryMTP`](../../circuits/main-circuits.md#credentialatomicquerymtp) based on the query.

### Verification of On-chain Identity States

Starting from the Identifier of the user, the [State](../../contracts/overview.md) is fetched from the blockchain and compared to the state provided as input to the proof to check whether the user is actually "owner" of the state used to generate the proof. It's important to note here is that there's no gas cost associated with the verification as the VerifyState method is just reading the identity state of the user on-chain without making any operations/smart contract call. The same verfication is performed for the Issuer Identity State.

In this part, it is also verified that the claim subject of the query hasn't been revoked by the Issuer.

### Verification of Circuit Public Inputs

This involves a verification based on the public inputs of the circuits used to generate the proof. These must match the rules requested by the verifier inside the auth request. For example the query and the claim schema used by the user to generate the proof must match the auth request:

  - The message signed by the user is the same as the one passed to the user in the auth request
  - The rules such as the `query` or the claim `schema` used as public input for the circuit match the ones included inside the auth request. 
  
This "off-circuit" verification is important because a user can potentially modify the query and present a valid proof. A user born the 2000-12-31 shouldn't pass the check. But if they generate a proof using a query input `"$lt": 20010101`, the verifier would see it as a valid proof. By doing verifying the public inputs of the circuit, the verifier is able to detect the cheat.

> At the end of the workflow:
> - The web-client is able to authenticate the user using its identifier `ID` after having established that the user controls that identity and satisfies the query presented in the auth request.
> - The user is able to log-in to the platform without disclosing any personal information to the client exept for its identifier
