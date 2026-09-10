
# API design
## Clear naming
- instead of https://example.com/api/v1/cart/123 go for https://example.com/api/v1/carts/123
/carts => collections

## idempotent API
- GET - idempotent
- POST - not naturally
- PUT - dempotent
- PATCH - not naturally
- DELETE
## Add versioning
- 
## Add pagination
- page + Offset
- Cursor-based
## clear query strings for sorting and filtering API data
- GET /users?sort_by=registered
- GET /products?filter=color:blue
## Security
- for sensitive credentials use Header, not URL. URLs are in server access log
- enformce TLS Encryption
- Access control
## Plan for rate limiting
- 