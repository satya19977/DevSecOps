# DevSecOps
## This is an end-to-end implementation of security practices across the Application(OWASP Juice Shop), Infrastructure (AWS) and Platform (Kubernetes)

<details>
<summary><h2>1 - Application Security</h2></summary>

<details>
<summary><h3>1.1 - Application Vulnerability Scanning </h3></summary>

<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>    




<details>
<summary><h3>1.2 - Vulnerability Management and Remediation  </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>


<details>
<summary><h3>1.3 - Vulnerability Scanning for Application Dependencies   </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>


<details>
<summary><h3>1.4 - Build a CD Pipeline   </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>

<details>
<summary><h3>1.5 - Image Scanning - Build Secure Docker Images    </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>

</details>



<details>
<summary><h2>2 - Infrastructure Security</h2></summary>

(Add Infrastructure Security content here)
</details>
<details>
    
<summary><h2>3 - Platform Security</h2></summary>

(Add Platform Security content here)
</details>





