# PR Agent

## Setup shared secret token using in both gitlab and pr agent
SHARED_SECRET=$(python -c "import secrets; print(secrets.token_hex(10))")

creating bot user for pr agent and giving it reporter role

webhook:
Secret Token : a shared secret token that verifies the authenticity of the webhook request


