# Supabase Authentication System Template

## SMS OTP with N8N

### Supabase Cloud

- [ ] Create new project
- [ ] Go to **Authentication** &rarr; **Auth Hooks** &rarr; **Add new _Send email hook_**
  - [ ] Choose HTTPS
  - [ ] Enter N8N Webhook [Supabase WhatsApp Connector](#supabase-whatsapp-connector-workflow)
  - [ ] Enter _Webhook Signature Secret_ or Generate new one by click **Generate secret**
  - [ ] Click **Create hook**
- [ ] Go to **Authentication** &rarr; **Sign In / Provider** and enable _Phone Provider_

### Self Hosted

- [ ] Add this environment variable when start supabase
  ```env
  GOTRUE_EXTERNAL_PHONE_ENABLED=true
  GOTRUE_HOOK_SEND_SMS_ENABLED=true
  GOTRUE_HOOK_SEND_SMS_URI=**********************
  GOTRUE_HOOK_SEND_SMS_SECRETS=v1,whsec_************************
  ```
 
## Supabase WhatsApp Connector Workflow

## SaaS Setup
