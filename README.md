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

- [ ] Create function `update_updated_at_column`
  ```sql
  CREATE OR REPLACE FUNCTION public.update_updated_at_column()
  RETURNS TRIGGER AS $$
  BEGIN
      NEW.updated_at = now();
      RETURN NEW;
  END;
  $$ LANGUAGE plpgsql SET search_path = public;
  ```
- [ ] Create table `companies`
  ```sql
  CREATE TABLE companies (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    description TEXT NULL,
    metadata JSONB DEFAULT '{}'::JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
  );

  CREATE TRIGGER update_companies_updated_at BEFORE
  UPDATE ON companies FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
  ```
- [ ] Create table `profiles`
  ```sql
  CREATE TABLE profiles (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    first_name TEXT NULL,
    last_name TEXT NULL,
    email TEXT NULL UNIQUE,
    avatar_url TEXT NULL,
    bio TEXT NULL,
    date_of_birth DATE NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
  );

  CREATE TRIGGER update_profiles_updated_at BEFORE
  UPDATE ON profiles FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

  CREATE OR REPLACE FUNCTION public.handle_new_user()
  RETURNS TRIGGER
  LANGUAGE plpgsql
  SECURITY DEFINER SET search_path = public
  AS $$
  BEGIN
    INSERT INTO public.profiles (user_id, phone)
    VALUES (NEW.id, NEW.phone);
    RETURN NEW;
  END;
  $$;

  CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
  ```
- [ ] Create table `permissions`
  ```sql
  CREATE TABLE permissions (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    resource TEXT NOT NULL,
    action TEXT NOT NULL,
    description TEXT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT permissions_resouce_action_key UNIQUE (resource, action)
  );
  ```
- [ ] Create table `roles`
  ```sql
  CREATE TABLE roles (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    description TEXT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
  );

  CREATE TRIGGER update_roles_updated_at BEFORE
  UPDATE ON roles FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
  ```
- [ ] Create table `role_permissions`
  ```sql
  CREATE TABLE role_permissions (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    fields JSONB NOT NULL DEFAULT '{}'::JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT role_permissions_role_id_permission_id_key (role_id, permission_id)
  );
  ```
- [ ] Create table `user_roles`
  ```sql
  CREATE TABLE user_roles (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    active BOOL NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT user_roles_user_id_role_id_company_id_key (user_id, role_id, company_id)
  );

  CREATE TRIGGER update_user_roles_updated_at BEFORE
  UPDATE ON user_roles FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
  ```
- [ ] Create function `has_permission`
  ```sql
  CREATE FUNCTION has_permission (
    user_id UUID,
    company_id UUID,
    resource TEXT,
    action TEXT,
    data jsonb default null
  )
  RETURNS boolean
  LANGUAGE sql
  SECURITY DEFINER
  SET search_path = public
  AS $$
    SELECT
      EXISTS (
        SELECT 1
        FROM public.user_roles ur
        JOIN public.role_permissions rp ON rp.role_id = ur.role_id
        JOIN public.permissions p ON rp.permission_id = p.id
        WHERE ur.user_id = has_permission.user_id
          AND ur.company_id = has_permission.company_id
          AND p.resource = has_permission.resource
          AND p.action = has_permission.action
          AND (
            rp.fields = '{}'::text[]  -- full access
            OR has_permission.data IS NULL
            OR NOT EXISTS (
              SELECT 1
              FROM jsonb_object_keys(has_permission.data) col
              WHERE col NOT IN (SELECT unnest(rp.fields))
            )
          )
        );
  $$;
  CREATE FUNCTION has_permission(
    user_id uuid,
    resource text,
    action text,
    data jsonb default null
  )
  RETURNS boolean
  LANGUAGE sql
  SECURITY DEFINER
  SET search_path = public
  AS $$
    SELECT
      EXISTS (
        SELECT 1
        FROM public.user_roles ur
        JOIN public.role_permissions rp ON rp.role_id = ur.role_id
        JOIN public.permissions p ON rp.permission_id = p.id
        WHERE ur.user_id = has_permission.user_id
          AND p.resource = has_permission.resource
          AND p.action = has_permission.action
          AND (
            rp.fields = '{}'::text[]  -- full access
            OR has_permission.data IS NULL
            OR NOT EXISTS (
              SELECT 1
              FROM jsonb_object_keys(has_permission.data) col
              WHERE col NOT IN (SELECT unnest(rp.fields))
            )
          )
        );
  $$;
  ```
- [ ] Create RLS policy
  - [ ] Companies
    ```sql
    ALTER TABLE companies ENABLE ROW LEVEL SECURITY;

    CREATE POLICY "User can create company when have permissions"
    ON companies AS PERMISSIVE FOR INSERT TO authenticated
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'companies', 'insert', TO_JSONB(companies.*))
    );

    CREATE POLICY "User can view their own companies when have permissions"
    ON companies AS PERMISSIVE FOR SELECT TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), companies.id, 'companies', 'view')
    );

    CREATE POLICY "User can update their own companies when have permissions"
    ON companies AS PERMISSIVE FOR UPDATE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), companies.id, 'companies', 'update')
    )
    WITH CHECK (
      has_permission((SELECT uid() AS uid), companies.id, 'companies', 'update', TO_JSONB(companies.*))
    );

    CREATE POLICY "User can delete their own companies when have permissions"
    ON companies AS PERMISSIVE FOR DELETE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), companies.id, 'companies', 'delete')
    );

    INSERT INTO permissions (resource, action, description)
    VALUES ('companies', 'insert', 'User can create company'),
           ('companies', 'view', 'User can view their own companies'),
           ('companies', 'update', 'User can update their own companies'),
           ('companies', 'delete', 'User can delete their own companies');
    ```
  - [ ] Profiles
    ```sql
    ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

    CREATE POLICY "User can create their own profile"
    ON profiles AS PERMISSIVE FOR INSERT TO authenticated
    WITH CHECK (
      (SELECT uid() AS uid) = profiles.user_id
    );

    CREATE POLICY "User can view their own profiles"
    ON profiles AS PERMISSIVE FOR SELECT TO authenticated
    USING (
      (SELECT uid() AS uid) = profiles.user_id
    );

    CREATE POLICY "User can update their own profiles"
    ON profiles AS PERMISSIVE FOR UPDATE TO authenticated
    USING (
      (SELECT uid() AS uid) = profiles.user_id
    )
    WITH CHECK (
      (SELECT uid() AS uid) = profiles.user_id
    );
    ```
  - [ ] Permissions
    ```sql
    ALTER TABLE permissions ENABLE ROW LEVEL SECURITY;

    CREATE POLICY "User can create permissions when have permissions"
    ON permissions AS PERMISSIVE FOR INSERT TO authenticated
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'permissions', 'insert', TO_JSONB(permissions.*))
    );

    CREATE POLICY "User can view permissions"
    ON permissions AS PERMISSIVE FOR SELECT TO authenticated
    USING (
      true
    );

    CREATE POLICY "User can update permissions when have permissions"
    ON permissions AS PERMISSIVE FOR UPDATE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), 'permissions', 'update')
    )
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'permissions', 'update', TO_JSONB(permissions.*))
    );

    CREATE POLICY "User can delete permissions when have permissions"
    ON permissions AS PERMISSIVE FOR DELETE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), 'permissions', 'delete')
    );

    INSERT INTO permissions (resource, action, description)
    VALUES ('permissions', 'insert', 'User can create permissions'),
           ('permissions', 'update', 'User can update permissions'),
           ('permissions', 'delete', 'User can delete permissions');
    ```
  - [ ] Roles
    ```sql
    ALTER TABLE roles ENABLE ROW LEVEL SECURITY;

    CREATE POLICY "User can create roles when have permissions"
    ON roles AS PERMISSIVE FOR INSERT TO authenticated
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'roles', 'insert', TO_JSONB(role.*))
    );

    CREATE POLICY "User can view roles"
    ON roles AS PERMISSIVE FOR SELECT TO authenticated
    USING (
      true
    );

    CREATE POLICY "User can update roles when have permissions"
    ON roles AS PERMISSIVE FOR UPDATE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), 'roles', 'update')
    )
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'roles', 'update', TO_JSONB(roles.*))
    );

    CREATE POLICY "User can delete roles when have permissions"
    ON roles AS PERMISSIVE FOR DELETE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), 'roles', 'delete')
    );

    INSERT INTO permissions (resource, action, description)
    VALUES ('roles', 'insert', 'User can create roles'),
           ('roles', 'update', 'User can update roles'),
           ('roles', 'delete', 'User can delete roles');
    ```
  - [ ] Role Permissions
    ```sql
    ALTER TABLE role_permissions ENABLE ROW LEVEL SECURITY;

    CREATE POLICY "User can create role_permissions when have permissions"
    ON role_permissions AS PERMISSIVE FOR INSERT TO authenticated
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'roles', 'insert', TO_JSONB(role_permissions.*))
      OR has_permission((SELECT uid() AS uid), 'roles', 'update', TO_JSONB(role_permissions.*))
    );

    CREATE POLICY "User can view role_permissions"
    ON role_permissions AS PERMISSIVE FOR SELECT TO authenticated
    USING (
      true
    );

    CREATE POLICY "User can delete role_permissions when have permissions"
    ON role_permissions AS PERMISSIVE FOR DELETE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), 'roles', 'update')
    );
    ```
  - [ ] User Roles
    ```sql
    ALTER TABLE user_roles ENABLE ROW LEVEL SECURITY;

    CREATE POLICY "User can create user_roles when have permissions"
    ON user_roles AS PERMISSIVE FOR INSERT TO authenticated
    WITH CHECK (
      has_permission((SELECT uid() AS uid), 'users', 'insert')
      OR has_permission((SELECT uid() AS uid), 'users', 'update')
    );

    CREATE POLICY "User can view their own user roles"
    ON user_roles AS PERMISSIVE FOR SELECT TO authenticated
    USING (
      (SELECT uid() AS uid) = user_roles.user_id
    );

    CREATE POLICY "User can delete user_roles when have permissions"
    ON user_roles AS PERMISSIVE FOR DELETE TO authenticated
    USING (
      has_permission((SELECT uid() AS uid), 'users', 'update')
    );
    ```
