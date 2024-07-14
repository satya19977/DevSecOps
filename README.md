# DevSecOps
This is an end-to-end implementation of security practices across the Application(OWASP Juice Shop), Infrastructure (AWS) and Platform (Kubernetes)

<details>
<summary><b>1 - Application Security</b></summary>
  
### Application Vulnerability Scanning

#### Q) Why do we need to integrate Security into our pipeline when we can delegate that to the Security team? It is faster to set up a basic pipeline, right?

Think again. These are the impacts of setting up basic CI Pipeline without security measures

We don't know ,

1) How secure our application is
2) If our code allows SQL injections
3) If there are any vulnerabilities for XSS
4) If there are any hardcoded credentials

#### Q) So what do we do then?

Solution: Integrate App Vulnerability Scanning Tools and  we want to add the security tests from the beginning, integrating it into the developer workflow instead of a separate isolated step

#### Q) How and where do we start?

Let's start with Secret Scanning

#### Q) Why do we need Secret Scanning?

We start with scanning our application for leaked secrets such as passwords, apikeys and any hardcorded credentials. 

Think of it this way. Having a leaky asset such as passwords and apikeys slipping into production is like taking your Instagram or Facebook credentials and publishing them over the internet. Anybody looking for it can access it and compromise your data.

#### Q) Ok, so what are secret scanning tools?

These are tools that scan your application code for leaked secrets

#### Q) What is the best secret scanning tool?

It depends on use case and what you are trying to achieve.

For it's reliability and ease of use, we go with Gitleaks. It detects over 160 types of secrets

Let's implement Gitleaks into our code and find out the vulnerabilities in our code

  ```
   stages:
    - cache
    - test
    - build

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

## This stage scans the code for sensitive information such as passwords, tokens

    ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands execute as intended.
gitleaks:
    stage: test
    image:
         name: zricethezav/gitleaks
         entrypoint: [""]
    ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
    script:
        - gitleaks detect --verbose . -f json -r gitleaks.json
    ## This is set to true because certain tokens in the jwt result in false positives which fail the build
    allow_failure: true

  ```

## Findings

```
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

```

#### *** As is evident from our findings above, we can see that our application code has Passwords, JWT Token and Api keys being exposed. This report also tells us which file and line number has the secret leaked from .***

#### Q) So that's it. We just have to implement these tools and our work is done?

No. Security is always layered and it has to implemented at every step. 

We have integrated Gitleaks in our CICD Piepeline but what if the credentials make their way into Repositories like Github? Any hacker can scan the commits and extract the information needed.

#### Q) So what do we do then?

Two options we can configure Gitleaks to scan the git or take the "Shift left" approach

#### Q) What is the Shift left approach?

It means to integrating security practices early in the software development lifecycle (SDLC) and as part of it we setup pre-commit hooks

#### Q) What is a  pre-commit hook?

Git Hooks is a Git functionality. It’s a way to fire off custom scripts when certain important actions occur

Pre-commit hooks automatically run scan before code is pushed to remote Git repository. With this we prevent any hard-coded secrets in the Git repository

#### Q) So we used gitleaks to scan our code for leaked secrets but how do we detect vulnerabilities in our code such as SQL Injection, XSS?

Detecting leaked secrets is just one part of vulnerability scanning

The code itself can be written in a way that allows for exploitation

To detect those vulnerabilities and help developers write secure code, we use SAST

#### Q) What is SAST?

It stands for Static Applications Seccurity Testing(app is not running). It identifies security vulnerabilities in app’s source code, configuration files etc.using SAST tools

There are SAST tools for different programming languages because each language has it's own specific syntax but there are tools that can scan multiple languages like SemGrep, SonarQube

These tools have different levels of severity such as Critical, medium , low or info. It helps us understand the level of risk posed by the vulnerability.

Multiple readings from different sources have suggested to combine different SAST tools for better code coverage. One tool is good at finding SQL Injection and the other one at say Path Traversal

</details>

<details>
<summary><b>1 - Infrastructure Security</b></summary>


</details>

<details>
<summary><b>1 - Platform Security</b></summary>


</details>
