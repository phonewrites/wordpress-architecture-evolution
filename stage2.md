1. SecureString type parameters cannot be created in Cloudformation. In production environment where budget allows, Secrets Manager should be used. To save costs, we create DBPassword & DBRootPassword parameters either manually in the console or by using the following commands in AWS CLI.
    - `aws ssm put-parameter --name "/Wordpress/DBPassword" --type SecureString --value "[REPLACE_WITH_PASSWORD_FOR_DBUser]" --description "Password for the DB user"`
    - `aws ssm put-parameter --name "/Wordpress/DBRootPassword" --type SecureString --value "[REPLACE_WITH_PASSWORD_FOR_DBRootUser]" --description "Password for the DB root user – used for self-managed admin"`

2. Later in UserData bootstrap script, we import the above values by using the `get-parameters` subcommand & `--with-decryption` flag for SecureString type parameters.

Reference: https://aws.amazon.com/blogs/mt/using-aws-systems-manager-parameter-store-secure-string-parameters-in-aws-cloudformation-templates/
