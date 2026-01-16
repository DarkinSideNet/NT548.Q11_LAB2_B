# Hướng dẫn sử dụng (CloudFormation + CodePipeline)

Dự án này triển khai hạ tầng AWS hoàn toàn bằng CloudFormation và tự động hóa qua CodePipeline/CodeBuild.

## Tổng quan
- CloudFormation [cloudformation/pipeline.yml](cloudformation/pipeline.yml) tạo:
	- S3 Artifact bucket
	- CodeCommit repository (nguồn)
	- CodeBuild project (lint CloudFormation)
	- CodePipeline (Source → Build → Deploy)
- Hạ tầng mục tiêu: [cloudformation/templates/infra.yml](cloudformation/templates/infra.yml)
	- VPC, Subnet Public/Private, IGW, NAT Gateway, Route Tables, Security Groups, EC2 Public/Private
- Tham số deploy: [cloudformation/templates/params-dev.json](cloudformation/templates/params-dev.json)

## Yêu cầu
- AWS CLI v2 và quyền tạo IAM, S3, CodeCommit, CodeBuild, CodePipeline, CloudFormation.
- Triển khai với `CAPABILITY_NAMED_IAM`.
- Region mặc định: `us-east-1`.

## Tạo Pipeline bằng CloudFormation
```bash
STACK_NAME=nt548-lab2-pipeline
REPO_NAME=nt548-lab2-repo
REGION=us-east-1

aws cloudformation deploy \
	--region "$REGION" \
	--stack-name "$STACK_NAME" \
	--template-file cloudformation/pipeline.yml \
	--capabilities CAPABILITY_NAMED_IAM \
	--parameter-overrides RepositoryName=$REPO_NAME BranchName=main
```

Push mã nguồn lên CodeCommit:
```bash
REPO_URL=$(aws codecommit get-repository --repository-name "$REPO_NAME" --query 'repositoryMetadata.cloneUrlHttp' --output text --region "$REGION")
git init
git remote add origin "$REPO_URL"
git add .
git commit -m "Initial commit"
git branch -M main
git push -u origin main
```

## Cấu hình tham số hạ tầng
Sửa [cloudformation/templates/params-dev.json](cloudformation/templates/params-dev.json):
- `KeyName`: tên EC2 Key Pair có sẵn
- `AmiId`: AMI hợp lệ cho region
- `AllowedSshCidr`: CIDR SSH vào EC2 public
- Có thể điều chỉnh CIDR VPC/Subnet, `InstanceType`.

## Quy trình Pipeline
- Source: lấy mã từ CodeCommit branch `main`.
- Build: CodeBuild chạy `cfn-lint` và `taskcat lint` trên template.
- Deploy: hành động CloudFormation `CREATE_UPDATE` dùng [cloudformation/templates/infra.yml](cloudformation/templates/infra.yml) và [cloudformation/templates/params-dev.json](cloudformation/templates/params-dev.json) để tạo/cập nhật stack.

## Tùy chọn nâng cao
- Thêm Manual Approval giữa Build và Deploy để kiểm soát triển khai sản xuất.
- Tạo nhiều file tham số (ví dụ `params-staging.json`, `params-prod.json`) và đổi `TemplateConfiguration` theo môi trường.

## Dọn dẹp
Xoá stack khi không cần:
```bash
aws cloudformation delete-stack --stack-name nt548-lab2-pipeline-infra --region us-east-1
```

Artifact bucket có thể cần xoá thủ công sau khi pipeline stack bị xoá.
