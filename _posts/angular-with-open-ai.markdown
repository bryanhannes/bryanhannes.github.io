---
layout: post
title:  "Let's build an Image Generator with OpenAI and Angular"
date:   2022-12-29 10:00:00 +0100
published: false
comments: false
categories: Angular
cover: assets/externalize-angular-configuration/externalize-configuration-angular-cover.png
---

# steps to set up project
# Create NX workspace
npx create-nx-workspace@latest angular-image-generation

# Add Angular app
npm install @nrwl/angular
nx generate @nrwl/angular:app client --routing --standalone --skipTests
-- choose SCSS

-- add skiptest to nx.json generators


# Adding express

npm i --save-dev @nrwl/express
nx generate @nrwl/express:app api

# Add openAI dependency
npm install openai --save

# Add logic to eslint
```json
  {
                "sourceTag": "type:app",
                "onlyDependOnLibsWithTags": ["type:feat", "type:util", "type:type"],
              },
              {
                "sourceTag": "type:type",
                "onlyDependOnLibsWithTags": [
                  "type:type"
                ],
              },
              {
                "type": "type:ui",
                "onlyDependOnLibsWithTags": [
                  "type:ui",
                  "type:util",
                  "type:type"
                ],
              },
              {
                "sourceTag": "type:feat",
                "onlyDependOnLibsWithTags": ["type:util", "type:type", "type:feat", "type:ui", "type:data-access"],
              },
              {
                "sourceTag": "type:util",
                "onlyDependOnLibsWithTags": ["type:type", "type:type"],
              },
              {
                "sourceTag": "type:data-access",
                "onlyDependOnLibsWithTags": ["type:util", "type:type", "type:data-access"],
              },
              {
                "sourceTag": "scope:client",
                onlyDependOnLibsWithTags: ["scope:client", "scope:shared"],
              },
              {
                "sourceTag": "scope:shared",
                onlyDependOnLibsWithTags: ["scope:shared"],
              }
```
# Set up shell feat libs

Add feat lib for the shell
```shell
npx nx generate @nrwl/angular:library client/feat-shell --tags=scope:client,type:feat 

```

Add smart component for the shell
```shell

```shell
npx nx g @nrwl/angular:component components/smart/shell --type=smart-component --project=client-feat-shell --dry-run 
```

# Add chat libs

```shell
npx nx generate @nrwl/angular:library chat/feat-chat --tags=scope:chat,type:feat
npx nx g @nrwl/angular:component components/smart/chat --type=smart-component --project=chat-feat-chat 
npx nx g @nrwl/angular:component components/ui/chat-history --type=ui-component --project=chat-feat-chat 

npx nx generate @nrwl/angular:library chat/data-access-chat --tags=scope:chat,type:data-access
npx nx g @nrwl/angular:service services/chat --project=chat-data-access-chat

npx nx generate @nrwl/angular:library chat/type-chat --tags=scope:chat,type:type
npx nx g @nrwl/angular:service services/chat --project=chat-type-answer


```

# Add image libs

```shell
npx nx generate @nrwl/angular:library image/feat-image --tags=scope:image,type:feat
npx nx g @nrwl/angular:component components/smart/image --type=smart-component --project=image-feat-image 

npx nx generate @nrwl/angular:library image/data-access-image --tags=scope:image,type:data-access
npx nx g @nrwl/angular:service services/image --project=image-data-access-image

npx nx generate @nrwl/angular:library image/type-image --tags=scope:image,type:type
npx nx g @nrwl/angular:service services/chat --project=chat-type-answer


```

# Add moderation libs

```shell
npx nx generate @nrwl/angular:library moderation/feat-moderation --tags=scope:moderation,type:feat
npx nx g @nrwl/angular:component components/smart/moderation --type=smart-component --project=moderation-feat-moderation 

npx nx generate @nrwl/angular:library moderation/data-access-moderation --tags=scope:moderation,type:data-access
npx nx g @nrwl/angular:service services/moderation --project=moderation-data-access-moderation

npx nx generate @nrwl/angular:library moderation/type-moderation --tags=scope:moderation,type:type

```

# Add UI lib

```shell
npx nx generate @nrwl/angular:library ui-library/feat-spinner --tags=scope:ui-library,type:feat
npx nx g @nrwl/angular:component components/spinner --type=ui-component --project=ui-library-feat-spinner

npx nx generate @nrwl/angular:library ui-library/feat-header --tags=scope:ui-library,type:feat
npx nx g @nrwl/angular:component components/header --type=ui-component --project=ui-library-feat-header

npx nx generate @nrwl/angular:library ui-library/util-vm --tags=scope:ui-library,type:util
npx nx g @nrwl/angular:component components/textarea --type=ui-component --project=ui-library-feat-textarea

```

# Add Tailwind CSS
```shell
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init
```


