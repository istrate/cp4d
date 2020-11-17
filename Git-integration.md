# To connect or not to connect

The GitHub integration with Watson Studio project. <br>
https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.0.1/wsj/manage-data/git-integration.html<br>

Sounds simple but not everything is obvious at the very first sight.

# Create GitHub personal access token

Upper-right image -> Settings -> Developer settings -> Personal access token

Use existing token or generate new one dedicated to IBM IP4D synchronization

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202020-11-17%2022-37-02.png)

# Register token in CP4D

A token is deployed by a logged person and is valid only for this person. More than one token can be added. "Token Name" is a description for the token.

Upper-right corner image -> Prfile and settings -> Git integration -> New Token

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202020-11-17%2022-49-28.png)

# Synchronization with IBM Watson Studio.

## Two scenarios

Two scenarios are possible. The first one is to create a new Watson Studio project and link it with empty GitHub repository. The second is to create a new Watson Studio project by importing from an existing project in GitHub repository. It is not possible to integrate the already created Watson Studio project with GitHub.

## New empty Watson Studio 

Create new GitHub repository. If repository is not empty, the content will be removed after integration with Watson Studio project.