---
title: An introduction to GitHub Actions
date: 2023-12-03 23:30:00 +0600
categories: [Guides]
tags: [cicd, github, github-actions, ecr, ecs-task-definition]
---

![Desktop View](/assets/img/bg/mike-benna-X-NAMq6uP3Q-unsplash.jpg){: w="700" h="400" }
_Photo by <a href="https://unsplash.com/@mbenna?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Mike Benna</a> on <a href="https://unsplash.com/photos/gray-pipe-on-green-grass-X-NAMq6uP3Q?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>_

GitHub Actions is free to use for standard GitHub-hosted runners in public repositories.

## Concepts

* Each job `runs-on` in a fresh virtual environment
* Each job consists of `steps`
* Each step `uses` another prebuilt action or can `run` some commands
* In an action file, each job runs in parallel by default. This can be prevented by using the `needs` param to refer to another job that needs to be completed first.
* Create `secrets` in your Github repository to pass sensitive values to the action file. You can put them in `Repository secrets` in Github console.

## Example: Push to ECR & revise ECS Task Definition

```yaml
on:
  push:
    branches:
      - main

name: Push image to ECR and revise ECS Task Definition

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ca-central-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: my-api-image
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG -f ./backend/Dockerfile ./backend
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Download task definition
        run: |
          aws ecs describe-task-definition \
          --task-definition MyAPITaskDef \
          --query taskDefinition > task-definition.json

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: MyAPIContainer
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: MyAPIService
          cluster: MyAPICluster
          wait-for-service-stability: true

```