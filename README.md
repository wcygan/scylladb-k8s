# ScyllaDB on Kubernetes

Reference: https://operator.docs.scylladb.com/stable/installation/helm.html

```bash
# If you don't have cert-manager in your cluster:
skaffold run -m cert-manager-config

# Install the ScyllaDB operator and deploy a cluster:
skaffold run -m scylla-operator-config
skaffold run -m scylla-config
skaffold run -m scylla-manager-config
```

Note: you will probably need to change the `storageClassName` in your Scylla [values.yaml](/scylla-config/values.yaml). I'm using `openebs-hostpath` which came with my Talos Linux installation.

If you already have cert-manager in your cluster, you can skip the first step.

Check for the pods to be ready:

```bash
kubectl -n scylla-operator get all
kubectl -n scylla get all
```

## Usage

```bash
# Get an interactive shell
kubectl exec -it scylla-dc1-rack1-0 -n scylla -- cqlsh

# Run a command through cqlsh
kubectl exec -it scylla-dc1-rack1-0 -n scylla -- cqlsh -e "DESC KEYSPACES"

# Use nodetool
kubectl exec -it scylla-dc1-rack1-0 -n scylla -- nodetool status
```

### Example Usage


1. First, let's connect to ScyllaDB:
```bash
kubectl exec -it scylla-dc1-rack1-0 -n scylla -- cqlsh
```

2. Create a keyspace with replication factor 3 (matching your 3-node cluster):
```sql
CREATE KEYSPACE IF NOT EXISTS user_management
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3
};
```

3. Switch to the new keyspace:
```sql
USE user_management;
```

4. Create a users table with good partitioning:
```sql
CREATE TABLE users (
    organization_id text,
    user_id uuid,
    email text,
    full_name text,
    created_at timestamp,
    last_login timestamp,
    is_active boolean,
    PRIMARY KEY ((organization_id), user_id)
);
```

5. Insert some sample data:
```sql
INSERT INTO users (
    organization_id,
    user_id,
    email,
    full_name,
    created_at,
    last_login,
    is_active
) VALUES (
    'org1',
    uuid(),
    'john.doe@example.com',
    'John Doe',
    toTimestamp(now()),
    toTimestamp(now()),
    true
);

INSERT INTO users (
    organization_id,
    user_id,
    email,
    full_name,
    created_at,
    last_login,
    is_active
) VALUES (
    'org1',
    uuid(),
    'jane.smith@example.com',
    'Jane Smith',
    toTimestamp(now()),
    toTimestamp(now()),
    true
);
```

6. Query examples:

Basic select:
```sql
SELECT * FROM users WHERE organization_id = 'org1';
```

Count users in organization:
```sql
SELECT COUNT(*) FROM users WHERE organization_id = 'org1';
```

Get specific user (you'll need an actual UUID):
```sql
SELECT * FROM users 
WHERE organization_id = 'org1' 
AND user_id = <uuid-from-previous-query>;
```

7. Create a secondary index for email queries:
```sql
CREATE INDEX IF NOT EXISTS users_email_idx ON users (email);
```

8. Query by email (using secondary index):
```sql
SELECT * FROM users WHERE email = 'john.doe@example.com' ALLOW FILTERING;
```

9. Update user data:
```sql
UPDATE users 
SET last_login = toTimestamp(now()), 
    is_active = false 
WHERE organization_id = 'org1' 
AND user_id = <uuid-from-previous-query>;
```

10. Delete a user:
```sql
DELETE FROM users 
WHERE organization_id = 'org1' 
AND user_id = <uuid-from-previous-query>;
```

11. Verify table structure:
```sql
DESCRIBE TABLE users;
```

12. Check consistency level (useful for troubleshooting):
```sql
CONSISTENCY;
```

Important notes:
- The PRIMARY KEY consists of a partition key (organization_id) and clustering key (user_id)
- This design allows efficient queries by organization
- Secondary indexes should be used sparingly in production
- ALLOW FILTERING should be avoided in production for large datasets
- Consider using prepared statements in your application code

To exit cqlsh:
```sql
exit;
```

To monitor the operation:
```bash
kubectl exec -it scylla-dc1-rack1-0 -n scylla -- nodetool tablestats user_management.users
```
