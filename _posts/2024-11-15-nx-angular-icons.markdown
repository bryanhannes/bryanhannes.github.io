---
layout: post
title:  "Generating icon components from SVG with NX and Angular"
date:   2024-11-15 05:00:00 +0100
published: false
comments: true
categories: Angular NX
cover: "assets/nx-angular-icons/nx-angular-icons"
tags: [Angular, NX]
type: article
---

TLDR: Find the source code on this [GitHub repository](https://github.com/bryanhannes/nx-angular-icons)

Generate 2 NX libraries (1 for the SVG files and 1 for the generated icons) and generate 1 NX plugin and a generator.
They need to separate libraries because the lib with the generator and SVG files will use CommonJS an the generated icons will use ES6 modules.

We assume you already have an NX workspace set up.

We placed the 2 libraries in a shared folder (with tags of `scope:shared` and `type:ui` and `type:tool`) but you can place them wherever you want.
The types will also depend on your own architecture.

Generating the ui-icons library:

```bash
npx nx g @nx/angular:library --name=shared-ui-icons --tags=scope:shared,type:ui --directory=libs/shared/ui-icons
```

Generating the icons source library:

```bash
npx nx g @nx/js:library --name=shared-tool-icon-generator --tags=scope:shared,type:tool --directory=libs/shared/tool-icon-generator --dry-run
```

Generating the NX plugin and generator
// first add nx/plugin in generate library for your custom NX generator in case you don't have it yet
```bash
npx nx add @nx/plugin
nx g @nx/plugin:plugin tools/nx-plugin 
```

Generating the icon generator
```bash
nx generate @nx/plugin:generator tools/nx-plugin/src/generators/icon-generator --name=icon-generator
```


/h1 The Icon Geneator
//h2 Optimizing the SVG files with SVGO

install SVGO

```bash
npm install -D svgo
```

Create a file called `svgo.config.js` in the `icon-generator` folder (`tools/nx-plugin/src/generators/icon-generator`) of the `shared-tool-icon-generator` library with the following content:

Update the settings in the `svgo.config.js` file to match your needs.
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

Replace all of the files of the `tools/nx-plugin/src/generators/icon-generator/files` folder with a file called: `__selector__.component.ts.template`

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

In case you want to add inputs to the icon component for size, color, etc. you can create a BaseIconComponent and then extend it in the generated icon components.
For simplicity in this article we will not add any inputs to the icon component.
But here is an example of how you could do it with a BaseIconComponent:

```typescript
// libs/shared/ui-icons/src/lib/base-icon.component.ts
export const iconSizes = ['s', 'm', 'l'] as const;

export type IconSize = (typeof iconSizes)[number];

@Component({
  template: '',
})
export abstract class BaseIconUiComponent {
  public size = input<IconSize>('m');

  @HostBinding('class')
  public get hostClasses(): string[] {
    return [`icon-${this.size()}`];
  }
}

// The selector component file would look like this:
import { ChangeDetectionStrategy, Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { BaseIconUiComponent } from '../base-icon.component'; // TODO Update path accordingly

@Component({
  selector: 'app-icon[<%= selector %>]', // TODO update the prefix according to your needs 
  standalone: true,
  template: `<%- svgCode %>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
  styleUrls: ['../icon.ui-component.scss']
})
export class <%= componentName %>Component extends BaseIconComponent { // Every generated Icon Component will extend from the BaseIconComponent
} 
```

 Add a new file here: `libs/shared/ui-icons/src/lib/icon.component.scss` with the following content:
```scss
:host {
  display: block;
  box-sizing: content-box;
  width: 24px;
  height: 24px
}

// In this file you would also add the extra styling for every size, color, etc.
```

Update the following file contents:

```typescript
// tools/nx-plugin/src/generators/icon-generator/generator.ts
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

export async function iconComponentGenerator(tree: Tree, options: IconComponentGeneratorSchema): Promise<void> {
  const projectRoot = `libs/shared/ui-icons/src/lib/generated`; // Update the path according to your needs

  const { iconPath, ...params } = options;
  const optimizedSvgContent = optimizeSvg(iconPath);

  generateFiles(tree, path.join(__dirname, 'files'), projectRoot, {
    ...params,
    svgCode: `${optimizedSvgContent}`,
  });

  await formatFiles(tree);
}

export default iconComponentGenerator;

// tools/nx-plugin/src/generators/icon-generator/schema.d.ts

export interface IconComponentGeneratorSchema {
componentName: string;
selector: string;
iconPath: string;
}
```

// `tools/nx-plugin/src/generators/icon-generator/schema.json`

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
    },
  },
  "required": ["selector", "iconPath", "componentName"]
}

```

The icons source folder and actual generator

Create a new folder called `icons-source` under `libs/shared/tools-icon-generator/src/lib` and place all the SVG files you want to create icons for in this folder.

In our example we 4 arrow SVG files in the `icons-source` folder.

Now we are going to create the actual script that is going to loop over all of SVG files in the `icons-source` folder and will call our custom NX generator to generate the icon components.

Create a file called `generate-icons.ts` in the `libs/shared/tool-icon-generator/src/lib` with the following content:

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
const generateAllIcons = args.find((arg) => arg.includes('--all'))?.split('=')[1] === 'true' ;

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
  return  kebabCaseString.split(/[-_]/).map((word) => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase()).join('');
}

function generateComponent(selector: string, componentName: string, svgPath: string): void {
  const command = `nx generate @nx-angular-icons/nx-plugin:icon-generator --componentName="${componentName}" --selector="${selector}"  --iconPath="${svgPath}" --quiet`;

  execSync(command, { stdio: 'inherit' });
}

function generateBarrelFile(icons: Map<string, string>): void {
  const exportStatements = Array.from(icons).map(([selector]) => `export * from './${selector}.component';`);

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

Now we want to add some ways to easily call this script,

we start by adding a new target to the project.json file of the `shared-tool-icon-generator` library:
// `libs/shared/tool-icon-generator/project.json`

```json
{
 ...
  "targets": {
    "generate": {
      "executor": "nx:run-commands",
      "options": {
        "commands": ["ts-node libs/shared/tool-icon-generator/src/lib/generate-icons.ts --all={args.all}"]
      }
    }
}
```

You can now run the script with `nx run shared-ui-icons-generator:generate` to generate only new icons or `nx run shared-ui-icons-generator:generate --all` to regenerate all icons.

A nice tip is to add these 2 scripts to your main `package.json` file: 
    
```json
    {
    "scripts": {
      "icons:generate": "nx run shared-tool-icon-generator:generate",
      "icons:generate-all": "nx run shared-tool-icon-generator:generate --all"
    }
}
```

// Testing it out

Now run to generate the icons 
```bash
npm run icons:generate

or

nx run shared-tool-icon-generator:generate
```

The output should look something like this:

```bash
> nx run shared-tool-icon-generator:generate

> ts-node libs/shared/tool-icon-generator/src/lib/generate-icons.ts --all=

✅ Start generating icon components
✅ Generated icon-arrow-up
✅ Generated icon-down-arrow
✅ Generated icon-left-arrow
✅ Generated icon-right-arrow
✅ Done generating icon components
✅ Generate barrel file (index.ts)
🏁 Icon components generated under: libs/shared/ui-icons/src/lib/generated/generated

—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

 NX   Successfully ran target generate for project shared-tool-icon-generator (8s)

```

In case you ever want to regenerate all icons you can run:

```bash
nx run shared-tool-icon-generator:generate --all
```

or 

```bash
npm run icons:generate-all
```

If you need to generate more icon components, simply add the SVG files to the `icons-source` folder and run the generate script again `npm run icons:generate`

We advise to put the generated icon components in Git so the CICD pipeline does not have to regenerate then everytime again. 

You can now use the generated icons in your Angular application like this:

```typescript
<app-icon[arrow-down] />
```

Or you can load them in dynamically with `ngComponentOutlet`:

```typescript
@Component({
  selector: 'app-my-example',
  template: `<ng-container *ngComponentOutlet="iconComponent()"></ng-container>`,
})
export class MyExampleComponent {
    iconComponent = signal(ArrowDownIconComponent);
}

```

That's it! You now have a fully automated way to generate icon components from SVG files with NX and Angular.

You can find the full source code here: [GitHub repository](https://github.com/bryanhannes/nx-angular-icons)

