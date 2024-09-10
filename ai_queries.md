This diagram illustrates the complete flow you described, including:
1. User navigating to the Angular App
2. Angular App checking authentication
3. Two paths:
   - Authenticated: making API calls directly
   - Unauthenticated: initiating OIDC flow
4. OIDC flow with Ping Identity:
   - Attempting to authenticate with the browser's logged-in user
   - Presenting a login interface if needed
   - Redirecting back to the Angular App with authorization code
5. Angular App exchanging authorization code for auth token with AuthServer
6. Angular App requesting ACL information from AuthServer
7. AuthServer returning entitlements and additional information
8. Angular App continuing to load UI and make API calls with obtained information

The diagram includes all the steps in the authentication and authorization process, showing the interactions between the user, browser, Angular App, Ping Identity, AuthServer, and API.



---

```mermaid
%%{init: {'theme': 'base', 'sequence': { 'showSequenceNumbers': true, 'actorMargin': 50, 'mirrorActors': false } } }%%
sequenceDiagram
    actor User
    participant Browser
    participant AngularApp
    participant PingIdentity
    participant AuthServer
    participant API

    User->>+Browser: Navigate to Angular App
    Browser->>+AngularApp: Load App
    AngularApp->>+AngularApp: Check Authentication

    alt User is Authenticated
        AngularApp->>+API: Make API Calls
        API-->>-AngularApp: API Response
    else User is Not Authenticated
        AngularApp->>+PingIdentity: Initiate OIDC Flow (client_id, redirect_uri, etc.)
        PingIdentity->>Browser: Attempt to Authenticate with Browser User
        
        alt Browser User Authentication Successful
            PingIdentity-->>AngularApp: Redirect to redirect_uri with Authorization Code
        else Browser User Authentication Failed
            PingIdentity->>Browser: Present Login Interface
            User->>Browser: Enter Credentials
            Browser->>PingIdentity: Submit Credentials
            
            alt Credentials Valid
                PingIdentity-->>AngularApp: Redirect to redirect_uri with Authorization Code
            else Credentials Invalid
                PingIdentity->>Browser: Show Error Message
            end
        end
        deactivate PingIdentity

        Note over PingIdentity,AngularApp: Authorization Code passed in query string

        AngularApp->>+AuthServer: POST to login endpoint (authorization_code, client_id, etc.)
        AuthServer-->>-AngularApp: Return Auth Token

        AngularApp->>+AuthServer: POST to ACL endpoint with Auth Token
        AuthServer-->>-AngularApp: Return Entitlements and Info in Response Headers

        AngularApp->>AngularApp: Continue loading UI
        AngularApp->>+API: Make API calls with Auth Token and Response Header Info
        API-->>-AngularApp: API Response
    end
    deactivate AngularApp
    deactivate Browser

```

---
