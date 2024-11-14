---
layout: post
title:  "Generating icon components from SVG files with NX and Angular"
date:   2024-11-13 05:00:00 +0100
published: true
comments: true
categories: Angular NX
cover: "assets/svg-to-icon-components/svg-to-icon-components"
tags: [Angular, NX]
type: article
---

# Generating Icon Components from SVG files with NX and Angular

## Overview

> **TLDR:** Find the complete source code in this [GitHub repository](https://github.com/bryanhannes/nx-angular-icons)

This guide will show you how to leverage NX to automatically convert SVG files into Angular components. You can follow along this guide or check out the [complete source code](https://github.com/bryanhannes/nx-angular-icons) to figure it out yourself. By the end of this tutorial, you'll have:

- A scalable system for managing SVG icons in your NX workspace
- Type-safe icon components with optimized SVG code

## Prerequisites

- An existing NX workspace
- Familiarity with Angular and NX

# Project structue overview
Before we start, let's understand the project structure we'll create:
```
your-nx-workspace/
├── libs/
│   └── shared/
│       ├── ui-icons/               # Generated icon components
│       │   └── src/
│       │       └── lib/
│       │           └── generated/  # Folder for auto-generated icon components
│       └── tool-icon-generator/    # Generator tool
│           └── src/
│               └── lib/
│                   ├── icons-source/  # Source SVG files
│                   └── generate-icons.ts
└── tools/
    └── nx-plugin/       
        └── src/
            └── generators/
                └── icon-generator/  # The actual generator
```

## Setting Up the Libraries

### Creating the Libraries

We'll create two separate libraries:

1. `shared-ui-icons`: An Angular library for our generated icon components
2. `shared-tool-icon-generator`: A JavaScript library for SVG sources and generation logic

We use an Angular library for the components and a plain JavaScript library for the generation tools since the latter doesn't require any Angular features.


#### Generate the libraries
```bash
# Generate the UI icons library
npx nx g @nx/angular:library --name=shared-ui-icons \
  --tags=scope:shared,type:ui \
  --directory=libs/shared/ui-icons

# Generate the icons source library
npx nx g @nx/js:library --name=shared-tool-icon-generator \
  --tags=scope:shared,type:tool \
  --directory=libs/shared/tool-icon-generator
```

### Setting Up the NX Plugin and Generator

The NX plugin will provide a custom generator that will be called from the `generate-icons.ts` script. This approach offers:
- Consistent icon component generation
- Built-in optimization

#### Add the NX plugin capability
```bash
# Only needed if you do not have an nx/plugin yet
npx nx add @nx/plugin
nx g @nx/plugin:plugin tools/nx-plugin
```

#### Create the icon generator
```bash
nx generate @nx/plugin:generator tools/nx-plugin/src/generators/icon-generator \
  --name=icon-generator
```

## Building the Icon Generator

### Optimizing SVG Files with SVGO

[SVGO](https://github.com/svg/svgo) (SVG Optimizer) is a Node.js tool that optimizes SVG files by removing redundant information and applying various transformations. It can significantly reduce SVG file size while preserving the visual output.

1. Install SVGO:
```bash
npm install -D svgo
```

2. Create `svgo.config.js` in the `tools/nx-plugin/src/generators/icon-generator` folder:

```javascript
module.exports = {
  plugins: [
    {
      name: 'preset-default',
      params: {
        overrides: {
          removeViewBox: false,
        },
      },
    },
    'removeDimensions',
    'cleanupAttrs',
    'removeXMLProcInst',
    'removeDimensions',
    'cleanupIds',
    'removeTitle',
    'removeUselessStrokeAndFill',
  ],
};
```

### Setting Up Component Templates

Create `__selector__.component.ts.template` in the `tools/nx-plugin/src/generators/icon-generator/files` folder:

```typescript
import { ChangeDetectionStrategy, Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-icon[<%= selector %>]',
  standalone: true,
  template: `<%- svgCode %>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
  styleUrls: ['../icon.ui-component.scss']
})
export class <%= componentName %>Component {
}
```

### Adding Base Styles

Create `libs/shared/ui-icons/src/lib/icon.component.scss`:

```scss
:host {
  display: block;
  box-sizing: content-box;
  width: 24px;
  height: 24px
}
```

### Implementing the Generator

Update the generator files with the following content:

1. `tools/nx-plugin/src/generators/icon-generator/generator.ts`:
```typescript
import * as path from 'path';
import { execSync } from 'child_process';
import { formatFiles, generateFiles, Tree } from '@nx/devkit';
import { IconComponentGeneratorSchema } from './schema';

function optimizeSvg(svgFilePath: string): string {
  const svgoConfigPath = path.join(__dirname, 'svgo.config.js');
  return execSync(`svgo --config="${svgoConfigPath}" --pretty --input=${svgFilePath} --output=-`, {
    encoding: 'utf8',
  });
}

export async function iconComponentGenerator(
  tree: Tree, 
  options: IconComponentGeneratorSchema
): Promise<void> {
  const projectRoot = `libs/shared/ui-icons/src/lib/generated`;

  const { iconPath, ...params } = options;
  const optimizedSvgContent = optimizeSvg(iconPath);

  generateFiles(tree, path.join(__dirname, 'files'), projectRoot, {
    ...params,
    svgCode: `${optimizedSvgContent}`,
  });

  await formatFiles(tree);
}

export default iconComponentGenerator;
```

2. `tools/nx-plugin/src/generators/icon-generator/schema.d.ts`:
```typescript
export interface IconComponentGeneratorSchema {
  componentName: string;
  selector: string;
  iconPath: string;
}
```

3. `tools/nx-plugin/src/generators/icon-generator/schema.json`:
```json
{
  "$schema": "http://json-schema.org/schema",
  "$id": "IconComponent",
  "title": "",
  "type": "object",
  "properties": {
    "selector": {
      "type": "string",
      "description": "",
      "$default": {
        "$source": "argv",
        "index": 0
      },
      "x-prompt": "What selector would you like to use?"
    },
    "iconPath": {
      "type": "string",
      "description": "",
      "$default": {
        "$source": "argv",
        "index": 0
      },
      "x-prompt": "What iconPath would you like to use?"
    },
    "componentName": {
      "type": "string",
      "description": "",
      "$default": {
        "$source": "argv",
        "index": 0
      },
      "x-prompt": "What componentName would you like to use?"
    }
  },
  "required": ["selector", "iconPath", "componentName"]
}
```

## Creating the icon generation script

1. Create a folder structure:
```
libs/shared/tools-icon-generator/src/lib/
└── icons-source/    # Place your SVG files here
```

2. Create `generate-icons.ts` in `libs/shared/tool-icon-generator/src/lib/`:
```typescript
import * as fs from 'fs';
import * as path from 'path';
import { execSync } from 'child_process';

const svgSourceFolderName = 'icons-source';
const svgSourceFolderPath = path.join(__dirname, svgSourceFolderName);

const targetFolderName = 'generated';
const targetBasePath = 'libs/shared/ui-icons/src/lib';
const targetFolderPath = path.join(targetBasePath, targetFolderName);
const barrelFilePath = path.join(targetBasePath, targetFolderName, 'index.ts');

/* Arguments */
const args = process.argv.slice(2);
const generateAllIcons = args.find((arg) => arg.includes('--all'))?.split('=')[1] === 'true';

/* Util functions */
function deleteFolderIfExists(folder: string) {
  if (fs.existsSync(folder)) {
    fs.rmSync(folder, { recursive: true, force: true });
  }
}

function createFolderIfItDoesNotExists(folder: string): void {
  if (!fs.existsSync(folder)) {
    fs.mkdirSync(folder, { recursive: true });
  }
}

function kebabToPascal(kebabCaseString: string): string {
  return kebabCaseString
    .split(/[-_]/)
    .map((word) => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
    .join('');
}

function generateComponent(selector: string, componentName: string, svgPath: string): void {
  const command = `nx generate @nx-angular-icons/nx-plugin:icon-generator --componentName="${componentName}" --selector="${selector}"  --iconPath="${svgPath}" --quiet`;
  execSync(command, { stdio: 'inherit' });
}

function generateBarrelFile(icons: Map<string, string>): void {
  const exportStatements = Array.from(icons).map(
    ([selector]) => `export * from './${selector}.component';`
  );
  fs.writeFileSync(barrelFilePath, `${exportStatements.join('\n')}`);
}

/* Main script */
if (generateAllIcons) {
  console.log(`✅ Deleting the output folder`);
  deleteFolderIfExists(targetFolderPath);
}

createFolderIfItDoesNotExists(targetFolderPath);

const sourceSvgFiles = fs.readdirSync(svgSourceFolderPath);
const icons = new Map<string, string>();

console.log(`✅ Start generating icon components`);

sourceSvgFiles.forEach((svgFile) => {
  const selector = svgFile.replace('.svg', '').toLowerCase();
  const fileExists = fs.existsSync(path.join(targetFolderPath, `${selector}.component.ts`));

  if (generateAllIcons || !fileExists) {
    const componentName = `${kebabToPascal(svgFile.replace('.svg', ''))}Icon`;
    const svgFilePath = path.join(__dirname, svgSourceFolderName, svgFile);

    generateComponent(selector, componentName, svgFilePath);
    console.log(`✅ Generated ${selector}`);
  } else {
    console.log(`✅ Did not regenerate ${selector}`);
  }

  const componentNameMatch = fs
    .readFileSync(path.join(targetFolderPath, `${selector}.component.ts`), 'utf8')
    .match(/export class (.*)/);

  if (componentNameMatch && componentNameMatch[1]) {
    icons.set(selector, componentNameMatch[1]);
  }
});

console.log(`✅ Done generating icon components`);
console.log(`✅ Generate barrel file (index.ts)`);
generateBarrelFile(icons);
console.log(`🏁 Icon components generated under:`, `${targetFolderPath}/${targetFolderName}`);
```

## Configuring build scripts

1. Update `libs/shared/tool-icon-generator/project.json`:
```json
{
  "targets": {
    "generate": {
      "executor": "nx:run-commands",
      "options": {
        "commands": [
          "ts-node libs/shared/tool-icon-generator/src/lib/generate-icons.ts --all={args.all}"
        ]
      }
    }
  }
}
```

2. **Optional**: Add convenience scripts to your root `package.json`:
```json
{
  "scripts": {
    "icons:generate": "nx run shared-tool-icon-generator:generate",
    "icons:generate-all": "nx run shared-tool-icon-generator:generate --all"
  }
}
```

## Usage

### Generating icons

To generate only new icons (those not already in the generated folder):

```bash
npm run icons:generate
# or
nx run shared-tool-icon-generator:generate
```

To regenerate all icons (useful after updating SVGO config or component templates):
```bash
npm run icons:generate-all
# or
nx run shared-tool-icon-generator:generate --all
```

### Using generated icons

The generated icons can be used in several ways in your Angular components:

```typescript
// 1. Static usage with selector
import { ArrowDownIconComponent } from '@your-workspace/shared/ui-icons';

@Component({
  // ...
  imports: [ArrowDownIconComponent],
  template: `<app-icon[arrow-down] />`
})
export class MyComponent {}

// 2. Dynamic usage with NgComponentOutlet
import { ArrowDownIconComponent, ArrowUpIconComponent } from '@your-workspace/shared/ui-icons';

@Component({
  // ...
  imports: [NgComponentOutlet, ArrowDownIconComponent, ArrowUpIconComponent],
  template: `<ng-container *ngComponentOutlet="currentIcon()"></ng-container>`
})
export class MyComponent {
  currentIcon = signal(ArrowDownIconComponent);
  
  // toggleIcon can be called from a button click or similar
  toggleIcon() {
    this.currentIcon.update(current => 
      current === ArrowDownIconComponent ? ArrowUpIconComponent : ArrowDownIconComponent
    );
  }
}
```

## Best Practices

### Version Control

1. **Commit generated icons**: Include generated icon components in your version control:
    - Prevents unnecessary regeneration during builds
    - Makes icon changes trackable through Git history
    - Ensures consistent icons across the team

2. **Review Changes**: When regenerating all icons (`--all` flag), review the Git diff to ensure changes are intentional.

### SVG source files 

Have naming conventions for the icons, in a large company this naming needs to be consistent between the UX team and the developers.
   ```
   action-name.svg        // e.g., arrow-down.svg
   object-name.svg        // e.g., shopping-cart.svg
   state-name.svg        // e.g., check-circle.svg
   ```
### Documentation
- Create an overview of available icons with Storybook for example.
- Add documentation for how to use the icon generator and generated components.

## Development Workflow

### Adding New Icons
   ```bash
   # 1. Add SVG file to icons-source/
   # 2. Generate only new icons
   npm run icons:generate
   # 3. Test the new icon
   # 4. Commit both source SVG and generated component
   ```

### Updating Existing Icons

   ```bash
   # 1. Replace SVG file in icons-source/
   # 2. Use --all flag to regenerate
   npm run icons:generate-all
   # 3. Test the updated icon
   # 4. Commit changes
   ```

## Wrapping up

That's all folks! This initial setup might take a bit of time, but it's worth it for the scalable, type-safe and consistent way of handling icons in your NX workspace.
Check out the complete source code on [Github](https://github.com/bryanhannes/nx-angular-icons).
