1. SecureString type parameters cannot be created in Cloudformation. In production environment where budget allows, Secrets Manager should be used. To save costs, we create DBPassword & DBRootPassword parameters either manually in the console or by using the following commands in AWS CLI.
    - aws ssm put-parameter --name "/Wordpress/DBPassword" --type SecureString --value "[REPLACE_WITH_PASSWORD_FOR_DBUser]" --description "Password for the DB user"
    - aws ssm put-parameter --name "/Wordpress/DBRootPassword" --type SecureString --value "[REPLACE_WITH_PASSWORD_FOR_DBRootUser]" --description "Password for the DB root user – used for self-managed admin"
2. Additionally, we need to create the following parameters as well.
    - aws ssm put-parameter --name "/Wordpress/DBName" --type String --value "wordpressdb" --description "Wordpress Database Name"
    - aws ssm put-parameter --name "/Wordpress/DBUser" --type String --value "wordpressuser" --description "Wordpress Database User"
    <!-- - aws ssm put-parameter --name "/Wordpress/DBEndpoint" --type String --value "localhost" --description "Wordpress Endpoint Name" -->

2. Later in UserData bootstrap script, we can import the above values by using the get-parameters subcommand & --with-decryption flag for SecureString type parameters.

3. Later in the RDS DB Instance resource properties, we can use the dynamic references to the above parameter values.

Reference: https://aws.amazon.com/blogs/mt/using-aws-systems-manager-parameter-store-secure-string-parameters-in-aws-cloudformation-templates/
