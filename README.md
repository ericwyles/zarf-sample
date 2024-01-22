This project is a simple spring boot web application I am using to see how to create and deploy zarf (https://zarf.dev) packages for java applications.

Initial app is adapted from https://github.com/binblee/springboot-helm-chart and then zarf configuration added.

I have not linked it to any CICD or github actions yet, but for local tinkering you can do the following:

## Build the docker image

```
cd demoweb
docker compose build
```
This should do a multi stage build and build the docker image for the application and put it in the local docker registry.

View the image using 'docker images' to see it as shown below:

```
REPOSITORY                        TAG                   IMAGE ID       CREATED          SIZE
ericwyles/springboot-helm-chart   jre-17                6af4dc3f15a0   10 minutes ago   288MB
```

There is a helm chart at demoweb/charts/springboot-demoweb/Chart.yaml that ultimately references the above image in the deployment.yaml as below

```
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

With values coming from values.yaml for the repository, tag and version:

```
image:
  repository: ericwyles/springboot-helm-chart
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "jre-17"

```


## Build the zarf package

After the docker image is built, run the following from the root of the repository

```
zarf package create
```

Answer the prompts and this will produce a zarf package called 'zarf-package-ewyles-spring-boot-demo-arm64-1.0.0.tar.zst' in the current directory.

### View SBOM

You can view the sbom by running 'zarf package inspect -s <package name>' and you'll get an output as shown below and it will launch the sbom viewer in your web browser:

```
 NOTE  Saving log file to
       /var/folders/sq/wmg9n0c549q4x4tw019bbrn80000gn/T/zarf-2024-01-22-17-34-00-3261916434.log
  ✔  Validating SBOM checksums                                                                                                                                       

kind: ZarfPackageConfig
metadata:
  name: ewyles-spring-boot-demo
  description: A Zarf Package that deploys a simple spring boot app
  version: 1.0.0
  architecture: arm64
  aggregateChecksum: 503b72395cb9bf3e09dc6d5ee7c47ff8525301122ed1b4e3499556b0823ce6cc
build:
  terminal: Erics-MacBook-Pro.local
  user: ericwyles
  architecture: arm64
  timestamp: Mon, 22 Jan 2024 17:32:53 -0600
  version: v0.32.1
  migrations:
  - scripts-to-actions
  - pluralize-set-variable
  lastNonBreakingVersion: v0.27.0
components:
- name: ewyles-spring-boot-demo
  description: Deploys the spring boot demo
  required: true
  charts:
  - name: ewyles-spring-boot
    version: 1.0.0
    localPath: demoweb/charts/springboot-demoweb/
    namespace: ewyles-spring-boot
    valuesFiles:
    - ewyles-spring-boot-demo-values.yaml
  manifests:
  - name: connect-services
    namespace: ewyles-spring-boot
    files:
    - connect-services.yaml
  images:
  - busybox
  - ericwyles/springboot-helm-chart:jre-17

 NOTE  This package has 2 images with software bill-of-materials (SBOM) included. If your browser
       did not open automatically you can copy and paste this file location into your browser
       address bar to view them:
       /var/folders/sq/wmg9n0c549q4x4tw019bbrn80000gn/T/zarf-2812086392/zarf-sbom/sbom-viewer-docker.io_ericwyles_springboot-helm-chart_jre-17.html
? Hit the 'enter' key when you are done viewing the SBOM files 
```


## Deploy the zarf package

Assuming you already have a kubernetes cluster with zarf initialized, run 'zarf package deploy <package name>'

It will print quite a bit of metadata and should finish with this:

```

? Deploy this Zarf package? Yes
  ✔  Waiting for cluster connection (30s timeout)                                                                                                                    
  ✔  Gathering additional cluster information (if available)                                                                                                         

                                                                                                      
  📦 EWYLES-SPRING-BOOT-DEMO COMPONENT                                                                
                                                                                                      

  ✔  Loading the Zarf State from the Kubernetes cluster                                                                                                              
  ✔  Pushed 2 images to the zarf registry                                                                                                                            
  ✔  Processing helm chart ewyles-spring-boot:1.0.0 from Zarf-generated helm chart                                                                                   
  ✔  Starting helm chart generation connect-services                                                                                                                 
  ✔  Processing helm chart raw-ewyles-spring-boot-demo-ewyles-spring-boot-demo-connect-services:0.1.1705966554 from Zarf-generated helm chart                        
  ✔  Zarf deployment complete

     Connect Command      | Description
     zarf connect sb-helm | The spring boot helm demo app
```

Running 'zarf connect sb-helm' will launch your web browser to the hello world page of the app with a tunnel created to the service:

```
ericwyles@Erics-MacBook-Pro zarf-sample % zarf connect sb-helm

 NOTE  Saving log file to
       /var/folders/sq/wmg9n0c549q4x4tw019bbrn80000gn/T/zarf-2024-01-22-17-37-20-552691975.log
  ⠇  Tunnel established at http://127.0.0.1:60959, opening your default web browser (ctrl-c to end) (12s)  
```

Or outside your browser you can curl the url printed above to see the output:

```
ericwyles@Erics-MacBook-Pro springboot-helm-chart % curl http://127.0.0.1:60959
Hello World.
```

The package now also shows up in 'zarf package list' in your local cluster

```
ericwyles@Erics-MacBook-Pro zarf-sample % zarf package list

 NOTE  Saving log file to
       /var/folders/sq/wmg9n0c549q4x4tw019bbrn80000gn/T/zarf-2024-01-22-17-43-54-2481246190.log
  ✔  Waiting for cluster connection (30s timeout)                                                                                                                    

     Package                 | Version | Components
     ewyles-spring-boot-demo | 1.0.0   | [ewyles-spring-boot-demo]
     init                    | v0.32.1 | [zarf-injector zarf-seed-registry zarf-registry zarf-agent logging git-server]
```


