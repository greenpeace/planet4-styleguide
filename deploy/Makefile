RELEASE_NAME ?= p4-styleguide

CHART_NAME ?= p4/static

SHELL := /bin/bash

NAMESPACE ?= default

DEV_CLUSTER ?= p4-development
DEV_PROJECT ?= planet-4-151612
DEV_ZONE ?= us-central1-a

PROD_CLUSTER ?= planet4-production
PROD_PROJECT ?= planet4-production
PROD_ZONE ?= us-central1-a

SED_MATCH ?= [^a-zA-Z0-9._-]
ifeq ($(CIRCLECI),true)
# Configure build variables based on CircleCI environment vars
BUILD_NUM = build-$(CIRCLE_BUILD_NUM)
BRANCH_NAME ?= $(shell sed 's/$(SED_MATCH)/-/g' <<< "$(CIRCLE_BRANCH)")
BUILD_TAG ?= $(shell sed 's/$(SED_MATCH)/-/g' <<< "$(CIRCLE_TAG)")
else
# Not in CircleCI environment, try to set sane defaults
BUILD_NUM = build-local
BRANCH_NAME ?= $(shell git rev-parse --abbrev-ref HEAD | sed 's/$(SED_MATCH)/-/g')
BUILD_TAG ?= $(shell git tag -l --points-at HEAD | tail -n1 | sed 's/$(SED_MATCH)/-/g')
endif



# If BUILD_TAG is blank there's no tag on this commit
ifeq ($(strip $(BUILD_TAG)),)
# Default to branch name
BUILD_TAG := $(BRANCH_NAME)
else
# Consider this the new :latest image
PUSH_LATEST := true
endif

REVISION_TAG = $(shell git rev-parse --short HEAD)


test:
	# yamllint .circleci/config.yml
	yamllint values.yaml
	yamllint env/dev/values.yaml
	yamllint env/prod/values.yaml

pull:
	docker pull gcr.io/planet-4-151612/openresty:latest

docker/checkout-develop:
	rsync -av ./ deploy/docker/source/ --exclude=deploy
	cd deploy/docker/source ; echo $CIRCLE_SHA1 > version.txt
	yq -r .hostname deploy/env/dev/values.yaml | tr -d '\n' > deploy/docker/source/hostname.txt
	yq -r .hostpath deploy/env/dev/values.yaml | tr -d '\n' > deploy/docker/source/hostpath.txt

docker/checkout-tag:
	git clone --depth 1 $(GIT_SRC) docker/source
	cd docker/source ; git checkout `git tag -l --points-at HEAD | tail -n 1`
	cd docker/source ; echo `git tag -l --points-at HEAD | tail -n 1` > version.txt
	yq -r .hostname env/prod/values.yaml | tr -d '\n' > docker/source/hostname.txt
	yq -r .hostpath env/prod/values.yaml | tr -d '\n' > docker/source/hostpath.txt

docker/public:
	cd docker/source ; npm install
	sudo npm install -g gulp-cli
	cd docker/source ; gulp build
	mv docker/source/dist docker/public

checkout-master: docker/checkout-master

checkout-tag: docker/checkout-tag

build: test pull docker/public echo-build
	docker build --tag=greenpeaceinternational/p4-styleguide:$(BUILD_NUM) docker

docker-push:
	docker push greenpeaceinternational/p4-styleguide:$(BUILD_NUM)

dev-config:
	gcloud config set project $(DEV_PROJECT)
	gcloud container clusters get-credentials $(DEV_CLUSTER) --zone $(DEV_ZONE) --project $(DEV_PROJECT)

prod-config:
	gcloud config set project $(PROD_PROJECT)
	gcloud container clusters get-credentials $(PROD_CLUSTER) --zone $(PROD_ZONE) --project $(PROD_PROJECT)

echo-build:
	$(info BUILD_NUM is [${BUILD_NUM}])

dev: test dev-config
	helm init --client-only
	helm repo add p4 https://planet4-helm-charts.storage.googleapis.com && \
	helm repo update
	helm upgrade --install --force --recreate-pods --wait $(RELEASE_NAME) $(CHART_NAME) \
		--namespace=$(NAMESPACE) \
		--set image.tag=$(BUILD_NUM) \
		--values values.yaml \
		--values env/dev/values.yaml

prod: test prod-config
	helm init --client-only
	helm repo add p4 https://planet4-helm-charts.storage.googleapis.com && \
	helm repo update
	helm upgrade --install --force --recreate-pods --wait $(RELEASE_NAME) $(CHART_NAME) \
		--namespace=$(NAMESPACE) \
		--set image.tag=$(BUILD_NUM) \
		--values values.yaml \
		--values env/prod/values.yaml
