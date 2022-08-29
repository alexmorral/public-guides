# How to use ImageMagick in AWS Lambda (ruby 2.7) with WebP support

## Table of contents

0. [Introduction](#0-introduction)
1. [Set up project](#1-set-up-project)
2. [Build and test Docker image](#2-build-and-test-docker-image)
3. [Store image automatically in AWS ECR using GitHub Actions](#3-store-image-automatically-in-aws-ecr-using-github-actions)
4. [Create AWS Lambda function](#4-create-aws-lambda-function)
5. [Conclusion](#5-conclusion)

## 0. Introduction

> **Context**: I wanted to create an AWS Lambda function that would be called from an app so I would send an image and the function would resize it, compress it (file size limitation) and send it back in WebP format.

Before even get into the guide itself, I want to say that before working on this architecture I tried to use the "layer" way: Creating an AWS Lambda layer with the ImageMagick dependency. But after wasting many hours not succeeding due to many issues, decided to switch to this "Docker" way.

I wanted to make this guide because while doing all of this process it's been really hard to find information. All the guides I found rely on this [layer repository](https://github.com/serverlesspub/imagemagick-aws-lambda-2) that it's not even updated anymore. I believe having the option to install whatever version of ImageMagick and libraries you want is better. So I hope this helps someone in the future.

## 1. Set up project
> **Documentation:** Create a Ruby image from an AWS base image
>
> https://docs.aws.amazon.com/lambda/latest/dg/ruby-image.html#ruby-image-create

The following will be the folder structure that we want.

```
my-project
-- app.rb
-- Gemfile
-- Dockerfile
```

### 1.1 Contents of `Gemfile`
To communicate from ruby to ImageMagick we will be using [RMagick](https://github.com/rmagick/rmagick), so we need to add it to our gems.

```ruby
source 'https://rubygems.org'
gem 'rmagick'
```

### 1.2 Contents of `app.rb`

The `app.rb` file will be our main ruby program. It will be executed by AWS Lambda so it needs to follow some namings.
```ruby
require 'rmagick'
include Magick

module LambdaFunction
    class Handler
        def self.process(event:, context:)
            image_url = event['image_url']
            my_image = ImageList.new(image_url)
            # TODO: Use rmagick to make your image transformations
            # Docs: https://rmagick.github.io
            { "success": true }
        end
    end
end   
```

### 1.3 Contents of `Dockerfile`

Oookay, here it comes the part where I spent most of the time. Lets start from the documentation `Dockerfile` and we'll add the commands needed.

```dockerfile
FROM public.ecr.aws/lambda/ruby:2.7

# Our commands will go here

COPY app.rb ${LAMBDA_TASK_ROOT}

COPY Gemfile ${LAMBDA_TASK_ROOT}

ENV GEM_HOME=${LAMBDA_TASK_ROOT}
RUN bundle install

CMD [ "app.LambdaFunction::Handler.process" ]
```

1. First of all we will install all the programs needed to build both the `libwebp` and ImageMagick libraries.

```dockerfile
RUN yum groupinstall -y 'Development Tools'
RUN yum install -y libmagickwand-devel libtool-ltdl-devel libpng-devel pkgconfig glibc ghostscript
```

2. Configure the `pkgconfig` path so when we configure ImageMagick with libwebp support, it finds the modules we built.

```dockerfile
ENV PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/lib/pkgconfig
```

3. Download and build `libwebp` library. This is because the Amazon ruby image doesn't come with an updated version of the libary. If we don't install an updated version, ImageMagick fails to configure WebP support on the next stage.

```dockerfile
RUN curl https://storage.googleapis.com/downloads.webmproject.org/releases/webp/libwebp-1.2.4.tar.gz --output libwebp-1.2.4.tar.gz \
    && tar xvzf libwebp-1.2.4.tar.gz\
    && cd libwebp-1.2.4 \
    && ./configure \
    && make \
    && make install \
    && /sbin/ldconfig /usr/local/lib
```

4. Download, configure and build ImageMagick. The code is mostly self-explanatory. We just configure ImageMagick with WebP support.

```dockerfile
RUN git clone https://github.com/ImageMagick/ImageMagick.git ImageMagick-7.1.0
RUN cd ImageMagick-7.1.0\
 && ./configure CPPFLAGS='-I/opt/local/include' LDFLAGS='-L/opt/local/lib' \
                --prefix=/usr \
                --without-perl \
                --with-modules \
                --without-magick-plus-plus \
		        --disable-static \
                --disable-docs \
                --with-png=yes \
                --with-webp=yes \
                --with-gslib=yes \
 && make\
 && make install\
 && /sbin/ldconfig /usr/local/lib
```

So the following is the final version of the `Dockerfile`:

```dockerfile
FROM public.ecr.aws/lambda/ruby:2.7 # OR: FROM amazon/aws-lambda-ruby:2.7

RUN yum groupinstall -y 'Development Tools'

RUN yum install -y libmagickwand-devel libtool-ltdl-devel libpng-devel pkgconfig glibc ghostscript

ENV PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/lib/pkgconfig

RUN curl https://storage.googleapis.com/downloads.webmproject.org/releases/webp/libwebp-1.2.4.tar.gz --output libwebp-1.2.4.tar.gz \
    && tar xvzf libwebp-1.2.4.tar.gz\
    && cd libwebp-1.2.4 \
    && ./configure \
    && make \
    && make install \
    && /sbin/ldconfig /usr/local/lib

RUN git clone https://github.com/ImageMagick/ImageMagick.git ImageMagick-7.1.0
RUN cd ImageMagick-7.1.0\
 && ./configure CPPFLAGS='-I/opt/local/include' LDFLAGS='-L/opt/local/lib' \
                --prefix=/usr \
                --without-perl \
                --with-modules \
                --without-magick-plus-plus \
		        --disable-static \
                --disable-docs \
                --with-png=yes \
                --with-webp=yes \
                --with-gslib=yes \
 && make\
 && make install\
 && /sbin/ldconfig /usr/local/lib

# Copy function code
COPY app.rb ${LAMBDA_TASK_ROOT}

# Copy dependency management file
COPY Gemfile ${LAMBDA_TASK_ROOT}

# Install dependencies under LAMBDA_TASK_ROOT
ENV GEM_HOME=${LAMBDA_TASK_ROOT}

RUN bundle install

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "app.LambdaFunction::Handler.process" ]
```

## 2. Build and test Docker image
> **Documentation:** Creating images from AWS base images: 
>
> https://docs.aws.amazon.com/lambda/latest/dg/images-create.html#images-create-from-base

Next step is building the Docker image, and everything should be working fine. 

I created a `Makefile`, because I'm lazy and I don't remember the commands:
```makefile
build:
	docker build --rm -t my-project .
run:
	docker run -p 9000:8080 my-project:latest
test:
	curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{"image_url": "https://www.cosy.sbg.ac.at/~pmeerw/Watermarking/lena_color.gif"}'
```

We already know what to do: `make build`, then `make run`. And from another terminal session execute `make test`.

## 3. Store image automatically in AWS ECR using GitHub Actions

Lets start by creating a repository in AWS Elastic Container Registry, where we will store our images. 

### Create AWS ECR Container 

In AWS Console (be sure to be in the right region):

1. Go to `Amazon Elastic Container Registry`

2. Create a new repository and set a name.

3. Now lets create a user to be able to access this repository. Go to `Identity Access Management (IAM)`

4. Users → Create new user → User with access key

5. Add the `AmazonEC2ContainerRegistryFullAccess` policy from the existing policies 
    > *We should create a proper policy to manage only your ECR repository, but that is not part of this guide.*

6. Save our Access Key and Secret Key.

### Create GitHub Action
 
Now lets create the GitHub action.

1. Go to your GitHub repository → Actions tab

2. Select `Set up a workflow yourself`

3. Add the following workflow changing the `ECR_REPOSITORY` and `IMAGE_TAG` to our project names. We'll also want to keep an eye on `aws-region` too, put the appropriate one in there.

```yml
name: Deploy to ECR
on:
  push:
    branches: [ master ]

jobs:
  build:
    name: Build Image
    runs-on: ubuntu-latest

    steps:
    - name: Check out code
      uses: actions/checkout@v2
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: eu-west-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-project-ecr-repository
        IMAGE_TAG: my-project-image
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

4. As you can see in the second step, we need to add the AWS Access and Secret keys to our GitHub repository. So in our repository, we go to `Settings` → `Secrets/Actions` and add the ones we saved previously when creating the user on IAM.
    - `AWS_ACCESS_KEY_ID`
    - `AWS_SECRET_ACCESS_KEY`

5. Now when we make a commit to our `master` branch we will trigger this Action and it will push the image to our ECR repository. Do that.

## 4. Create AWS Lambda function

Now we will create an AWS Lambda function based on this Docker image.

1. Go to AWS Lambda

2. Create new function → Container image

3. Add a name to our function and select the image from ECR

4. And that's it, we have our function created, we can test that everything works as expected.

> When we push new image versions to our ECR repository, the Lambda function **does not** update with the new version, we have to do it manually. I'm sure there is a way to auto-update it, but that's also not part of this guide.

## 5. Conclusion

There it is. Now we have an AWS Lambda function that has ImageMagick as a dependency.

I added this guide to a repository. PRs and feedback are really welcomed.

Repository: https://github.com/alexmorral/public-guides/blob/main/aws/lambda-imagemagick-docker.md
