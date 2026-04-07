Mac:thisisformula1-api rahuljangam$ aws ecr create-repository  --repository-name image-resizer  --region ap-southeast-2  --profile dev
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:ap-southeast-2:194694039009:repository/image-resizer",
        "registryId": "194694039009",
        "repositoryName": "image-resizer",
        "repositoryUri": "194694039009.dkr.ecr.ap-southeast-2.amazonaws.com/image-resizer",
        "createdAt": "2026-04-02T21:27:41.413000+11:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}

dockerfile      index.js        package.json
Mac:image-resizer rahuljangam$ docker tag image-resizer:latest 194694039009.dkr.ecr.ap-southeast-2.amazonaws.com/image-resizer:latest
Mac:image-resizer rahuljangam$ aws ecr get-login-password --region ap-southeast-2  --profile dev | docker login --username AWS --password-
stdin 194694039009.dkr.ecr.ap-southeast-2.amazonaws.com
Login Succeeded

Mac:image-resizer rahuljangam$ docker push 194694039009.dkr.ecr.ap-southeast-2.amazonaws.com/image-resizer:latest
The push refers to repository [194694039009.dkr.ecr.ap-southeast-2.amazonaws.com/image-resizer]

docker buildx build --platform linux/amd64 -t image-resizer .
docker tag image-resizer:latest 194694039009.dkr.ecr.ap-southeast-2.amazonaws.com/image-resizer:latest

docker push 194694039009.dkr.ecr.ap-southeast-2.amazonaws.com/image-resizer:latest