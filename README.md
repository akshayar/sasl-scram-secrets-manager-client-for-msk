This is a library intended to be used with Apache Kafka producers and consumers against an Amazon MSK Apache Kafka cluster 
utilizing SASL/SCRAM authentication, and administrative applications that manage secrets for SASL/SCRAM authentication. 
As explained in the [Amazon MSK documentation](https://docs.aws.amazon.com/msk/latest/developerguide/msk-password.html), 
Amazon MSK support for SASL/SCRAM authentication uses [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) to store 
usernames and passwords in secrets in AWS Secrets Manager and provides the ability to secure those secrets by encrypting 
them with Customer Master Keys (CMKs) from AWS Key Management Service (KMS) and attaching a Resource Policy to control 
who has access to the secret. 

For Apache Kafka producers and consumers to be able use the library to read the secrets from AWS Secrets Manager and 
configure the SASL/SCRAM properties, just the getSecretsManagerClient and the getSecret methods need to be used. All 
other methods are for managing secrets. Here is an example of code required to get a secret for Amazon MSK from 
AWS Secrets Manager that could be used in a producer or consumer.

```
String secretNamePrefix = "AmazonMSK_";
String saslscramUser = "nancy";
String region = "us-east-1"
String secret = Secrets.getSecret(secretNamePrefix + saslscramUser, Secrets.getSecretsManagerClient(region));
```

## Install

### Install the jar file.  

    mvn clean install -f pom.xml

# Supporting Password Roation
To support password rotation there are two problems to be solved. One change of password at MSK . 
Second managing authentication failure at client end and get new password on auth failure. 
For the first problem you need to integrate Secret Manager with a rotation function created usign AWS Lambda. 
Refer https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html#OTHER_rotation_templates
For the second problem you need to extend AuthenticateCallbackHandler and pull the secret again from the secret manager. 

## Rotation function using Lambda
When you create the Secret Manager configuration you can add rotation configuraiton and attach the a Lambda function. 
Secret Manager will use the Lambda function to change the password. 
For MSK the sample Lambda function is at [rotation-lambda.py](src/main/python/msk-sasl-scram-rotation-lambda/rotation-lambda.py)


## AuthenticateCallbackHandler
The clients have to handle authentication failure due to change of password and re-establish the connection and trust. 
Refer to the configuration at [kafka-sasl-secretmanager.properties](src/main/resources/kafka-sasl-secretmanager.properties).
Following is the relevant part. 
```
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  secretId="<ARN>" \
  region="<REGION>" ;
sasl.client.callback.handler.class=com.amazonaws.kafka.samples.saslscram.SecretManagerClientCallbackHandler

```
Refer the code for [com.amazonaws.kafka.samples.saslscram.SecretManagerClientCallbackHandler](src/main/java/com/amazonaws/kafka/samples/saslscram/SecretManagerClientCallbackHandler.java)