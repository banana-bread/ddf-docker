### Preparation
Creates a new AWS EC2 key pair with the name "ddf-adriano", retrieves the private key content, and saves it to a local file at ~/.ssh/ddf-adriano.pem.  This key pair allows us to securely connect to and authenticate EC2 instances and other AWS services that require SSH or PEM-based authentication.
```
aws ec2 create-key-pair --key-name ddf-adriano --query 'KeyMaterial' --output text > ~/.ssh/ddf-adriano.pem
```

Get more info about the key pair we just created
```
aws ec2 describe-key-pairs --key-name ddf-adriano
```

Now we'll create the security group, ddf_SG_useast1, to define inbound and outbound access rules for our EC2 instances and other related services in the us-east-1 region. This ensures controlled and secure access to our AWS resources.
```
aws ec2 create-security-group --group-name ddf_SG_useast1 --description "Security group for ddf on us-east-1"
```

Hold onto the `GroupId` output from the last command, we'll need this later.

Get more info about the security group we just created
```
aws ec2 describe-security-groups --group-id <group-id>
```

Now we need to modify our security group to allow inbound traffic through TCP protocol on port 22 (SSH) from any IP address, effectively enabling SSH access to our EC2 instances associated with our security group from anywhere on the internet
```
aws ec2 authorize-security-group-ingress --group-id <group-id> --protocol tcp --port 22 --cidr 0.0.0.0/0
```

Now, lets do the same thing for port 80, which will allow inbound http traffic
```
aws ec2 authorize-security-group-ingress --group-id <group-id> --protocol tcp --port 80 --cidr 0.0.0.0/0
```

We're also going to open up two more ports, for Amazon Elasticache and RDS.  These are managed AWS services for Redis and Postgres respectively.  The difference here is that we don't want to open these ports up to the entire internet.  We only want those ports to be accessible by our security group.  To achieve this, run the following command to open up port 6379 for Redis.
```
aws ec2 authorize-security-group-ingress --group-id <group-id> --protocol tcp --port 6379 --source-group <group-id>
```

Now run the same command, changing port to 5432 for Postgres
```
aws ec2 authorize-security-group-ingress --group-id <group-id> --protocol tcp --port 6379 --source-group <group-id>
```

### Create an s3 bucket
We'll use this to store config for ecs and the static build files of our React app
```
aws s3api create-bucket --bucket adriano-ddf
```

### Setting up RDS for Postgres
RDS is a managed solution for relational databases.  It is 25% more expensive than self-hosting our our ec2 instance, but probably worth it for a high traffic system

This command creates the database with some setup configs
```
aws rds create-db-instance --engine postgres --no-multi-az --no-publicly-accessible --vpc-security-group-ids <group-id> --db-instance-class db.t3.micro --allocated-storage 20 --db-instance-identifier ddf-production --db-name ddf_production --master-username <username> --master-user-password <password> --engine-version 15 
```

### Setting up Elasticache for Redis
Elasticache is a managed solution for Redis / memcached.  We can use this instead of managin our own EC2 instance.  Like RDS, Elasticache is 25% expensive than managing your EC2 instance, but worth it.

This command instructs AWS to create a new single-node Redis cache cluster with the identifier ddf-production using a t2.micro instance type and associates it with our security group.
```
aws elasticache create-cache-cluster --engine redis --security-group-ids <group-id> --cache-node-type cache.t2.micro --num-cache-nodes 1 --cache-cluster-id ddf-production
```

to get redis endpoint
```
aws elasticache describe-cache-clusters --show-cache-node-info
```

if you wanted to delete the cache cluster
```
aws elasticache delete-cache-cluster --cache-cluster-id ddf-prodction
```

### Setting up Elastic Load Balancer

todo: understand the realationship between subnets, ec2 intances, and elb.  was told to run this command to see subnets, then copy all the `SubnetId`'s where `DefaultForAz` is set to true.
```
 aws ec2 describe-subnets
```

Here, we are creating a new internet-facing Classic Load Balancer named ddf-api that listens on port 80 and routes traffic to backend instances on their port 80. In this case, our rails api. It will be created across multiple subnets (spanning multiple AZs) and will use the specified security group for its traffic rules
```
aws elb create-load-balancer --load-balancer-name ddf-api --listeners "Protocol=HTTP,LoadBalancerPort=80,InstanceProtocol=HTTP,InstancePort=80" --subnets <subnet-1> <subnet-2> <subnet-3>  --security-groups <group-id>
```

We can use this command to get more info on our newly created load balancer, like the subnets it operates in, availability zones, etc.
```
aws elb describe-load-balancers
```

Here, we are configuring the load balancers timeout to be 5 seconds.  5 seconds is an acceptable amount of time to receive a response in a normal web application.  If we have actions that take longer than this, say, zipping and downloading files, we would want to do this on a separate background process/worker
```
aws elb modify-load-balancer-attributes --load-balancer-name ddf-api --load-balancer-attributes "{\"ConnectionSettings\":{\"IdleTimeout\":5}}"
```

If we want to update the healthcheck endpoint
```
aws elb configure-health-check --load-balancer-name ddf-api --health-check "Targeet=HTTP:80/health_check,Timeout=5,Interval=30,UnhealthyThreshold=2,HealthyThreshold=10"
```

If we wanted to delete our load balancer
```
aws elb delete-load-balancer --load-balancer-name ddf-api
```