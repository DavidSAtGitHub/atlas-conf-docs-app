
## Basic Auth
- credentials (username + pass) encoded  and sent with request
- in not encryted with HTTPS, anyone could decode and use

## Digest Authentication
- Challenge response mechanism
- instead of sending PASS, server 1st. sends challenge to client
	- Client generates a hashed response using a password and challenge value and sends it back
	- Server verifies the hash instead of checking raw password
- prevents sending password in raw form
- not widely used because tokens are easier to scale

## Session auth
- lo manage login state
- when user logs in, server creates a session ID, stores it and sends to browser (cookie)
- with every other request, browser sends sessionID with it
- server needs to store session data, which is hard to scale on multiple servers

## API key
- simplest token mechanism
- API key is unique key assigned to each user/developer.   ... usually an app instead of individual user
- it's included in every request, so server knows, which app is calling 
- commonly used for public APIs - maps, weather, payment, ...
- often kombined with other auth mechanisms

## Bearer Token
- client sends a token in the authorisation header: bearer
- `whoever holds to token is allowed to use it 
- they are opaque, just random string - every single request checks DB for who token owner is?
- server verifies the token and allows aceess to the resource - doesn't need to store session state
- however it must be protected carefully - anyone can use it until it expires
- JWT contains everything server need:
- implemented as **JWT = token format** (header-payload-signature), digitaly signed by the server
	- signiture = header+payload both base64 hashed + secret_key encryption

### Access vs Refresh tokens
- Access - to access protected APIs, shortlived
- Refresh - longer lived, used to request new access token when old one expires

## OAuth2.0
- authorisation framework design to access resources on behalf of user without ever seeing the users password
- When app asks to access google drive/git hub ... user authenticates with external provider and app receives ACCESS token to grant limited permisions
- OA focuses on authorisation ... what is app allowed to access
- doesn't provide a standardised way to verify users identity

## OpenID Connect - OIDC
- builds on top of OAuth and adds  an identity layer
- ID token - contains verified information about user
- it allows apps to confirm users idetity without managing password itselves
- sign-in with GOOGLE

## SSO
- user signs in once and gain 
- SAML or OIDC makes it possible



## Opaque vs JWT
- opaque requires DB lookup, JWT just server decode 
- for JWT, every independent server can verify token 
- Opaque - delete token from DB  = no access
- JWT  - can't be recalled, so it lives for few minutes
	- SOLVED by using long lived tokens
	- REFRESH token is OPAQUE
## Signing algorithms
### Symetric
- HS256
- same secret key signs and verifies
- best for monolith - one server does everything
### Asymetric
- RS256
- Private key signs, public key verifies
