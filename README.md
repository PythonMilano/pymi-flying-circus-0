# PyMI Flying Circus #0

Integrazione di una soluzione single sign on cross language

TLDR: utilizzo di un software per l'SSO con un paio di integrazioni in Java e Python. Verranno introdotti i concetti base di un SSO, che cos'é e a cosa serve. Il software utilizzato è Keycloak, scritto in Java, sviluppato da Red Hat.

## Cos'è il Single Sing-On

Il Single sign-on (SSO) è la modalità che un utente ha per essere autenticato una volta e accedere a differenti applicazioni senza dover effettuare la login di nuovo.

L'attore che si occupa di riconoscere gli utenti si chiama identity provider (IdP)

In altre parole si può dire che all'IdP viene delegato il compito di autenticare l'utente evitando che reinserisca di volta in volta le credenziali per accedere alle applicazioni

L'IdP evita che le credenziali siano esposte alle applicazioni.

L'uso di un IdP ha un'unico punto di accesso con un solo form di login per tutte le applicazioni. Inoltre, può essere un unico punto di amministrazione per le regole di autenticazione. Le sessioni dovrebbero essere configurate per ciascuna applicazione che accede all'IdP.

## SAML2

Security Assertion Markup Language (SAML) è uno standard per lo scambio di dati di autenticazione e autorizzazione (dette assertions) tra domini di sicurezza distinti, tipicamente un identity provider (IdP) e un service provider (SP, entità che fornisce servizi). Il formato delle asserzioni SAML è basato su XML. SAML è mantenuto da OASIS Security Services Technical Committee.

## Software utilizzato

### Keycloack

> Open Source Identity and Access Management

Link: https://www.keycloak.org/

In Keycloak viene cofigurato un "reame" per identificare il campo di applicazione dell'IdP.

Nel reame vengono configurati diversi SP per ciascuna applicazione e ne permette l'interfacciamento con l'IdP.

Keycloack è altamente configurabile sia nei flussi di autenticazione che per i singoli SP.

Ad esempio si possono avere differenti flussi di autenticazione per i vari SP, pur mantenendo la stessa base dati utente.

## JWT

JSON Web Tokens è uno standard aperto identificato dalla RFC 7519 e serve per scambiare in modo sicuro dati tra le applicazioni.

### Python

Librerie

* [djangosaml2](https://pypi.org/project/djangosaml2/)
* [PySaml2](https://pypi.org/project/pysaml2/)
* [pyJWT](https://pypi.org/project/PyJWT/)

### Java

Implementazione di uno UserStorage all'interno di Keycloak.

https://www.keycloak.org/docs/latest/server_development/#_auth_spi_walkthrough

https://github.com/keycloak/keycloak-quickstarts/tree/latest/user-storage-simple

Implementazione di un flusso custom chiamando un'applicazione esterna.

https://github.com/keycloak/keycloak-quickstarts/tree/latest/action-token-authenticator

La parte interessante da capire è la creazione del token che permette di ritornare nel flusso principale. 

A differenza dell'esempio, qui viene creato un nuovo token e rilasciato per l'applicazione esterna.

    int validityInSecs = context.getRealm().getActionTokenGeneratedByUserLifespan();
    int absoluteExpirationInSecs = Time.currentTime() + validityInSecs;
    AuthenticationSessionModel authSession = context.getAuthenticationSession();
    String clientId = authSession.getClient().getClientId();

    String accessCode = context.generateAccessCode();
    // Create a token used to return back to the current authentication flow
    String appToken = new ExternalAppActionToken(currentUser.getId(), absoluteExpirationInSecs, clientId)
            .serialize(context.getSession(), context.getRealm(), context.getUriInfo());

    // This URL will be used by the application to submit the action token above to return back to the flow
    URI action = context.getActionUrl(accessCode);
    String submitActionTokenUrl = UriBuilder.fromUri(action)
            .queryParam(APP_TOKEN, appToken)
            .build()
            .toString();

Il token `appToken` è generato come indicato nella documentazione https://www.keycloak.org/docs/latest/server_development/#how-to-create-an-action-token

Nel codice viene poi utilizzato nel metodo `action` al ritorno dall'applicazione esterna.
Semplificando un po' il codice dell'esempio: 

    @Override
    public void action(AuthenticationFlowContext context) {
        // ...
        try {
            JsonWebToken appToken = TokenVerifier.create(appTokenString, JsonWebToken.class).getToken();
            final String appId = applicationId;
            appToken.getOtherClaims()
                    .forEach((key, value) -> user.setAttribute(appId + "." + key, Collections.singletonList(String.valueOf(value))));
        } catch (VerificationException ex) {
            // ...
        }
        context.success();
    }

Viene utilizzato JWT per lo scambio di dati. Il token è `signed`.

### Python e JWT

All'interno dell'applicazione esterna possiamo avere un token da utilizzare per avere differenti dati come campo del `CredentialModel` custom di Keycloak.

Questa modalità è servita per disaccoppiare le due applicazioni e non andare ad appesantire di campi il modello custom.

    user_data = {...}
    token = jwt.encode(user_data, dj_settings.KEYCLOAK_DATA_SIGNATURE, algorithm='HS256')
    data = {
        'account_id': account.id,
        'username': account.username,
        'first_name': account.first_name,
        'last_name': account.last_name,
        'email': account.email,
        'appToken': token,
    }


## Libro consigliato

[Solving Identity Management in Modern Applications: Demystifying OAuth 2.0, OpenID Connect, and SAML 2.0](https://www.apress.com/it/book/9781484250945#otherversion=9781484250952)


