---
layout: post
title:  "How to Call the OpenAI API Directly from Angular (with streaming)"
date:   2023-05-15 05:00:00 +0100
published: true
comments: true
categories: Angular OpenAI
cover: "assets/openai-api-and-angular/openai-api-and-angular"
tags: [Angular, OpenAI]
type: article
---

Did you know that you can call the OpenAI API directly from Angular? In this article, we'll cover how to call the [Chat Completions endpoint of the OpenAI API](https://platform.openai.com/docs/guides/chat){:target="_blank"}. Yes, we also included streaming, just because we can. You can use this code to create your own ChatGPT.

> Are you curious how to create your own ChatGPT? 

I created an Open Source version called [AmelioratedChat](https://amelioratedchat.com){:target="_blank"}. It's like ChatGPT, but
- with a better UI
- completely Open Source (You have to provide your own API key though 😅)
- with a ton of best practices 
- written with Angular and NX

You can find the source code from AmelioratedChat on [GitHub](https://github.com/bryanhannes/ameliorated-chat){:target="_blank"}.

But now, on to the actual article 👇

## Creating the OpenAI API service
The first step is to create an Angular service that will call the OpenAI API. 

This service has a single method `doOpenAICall`.
What this method does is, create an observable, that uses an XMLHttpRequest to call the OpenAI API. The OpenAI API will return a stream of updates (streaming), which we'll parse (extract the generated text = chunk) and emit to the observer. In the case the API returns a success status code (200), we complete the observable otherwise we error out.
The reason why we use XMLHttpRequest is that it supports streaming, unfortunately streaming is not supported by the Angular HttpClient. See [this issue](https://github.com/angular/angular/issues/44143#issuecomment-1163249379){:target="_blank"}.


```typescript

export type Role = 'user' | 'system' | 'assistant';

// Message type that represents a message in the conversation
export type Message = {
    content: string;
    role: Role;
};

@Injectable({ providedIn: 'root' })
export class OpenAIService {
  /**
    This Angular service method makes a call to the OpenAI API to perform a chat completion.
    It sends a list of messages along with other parameters to the API and receives responses in a streaming manner.
    @param messages - An array of Message objects representing the conversation history.
    @param temperature - Optional parameter that controls the randomness of the generated text. Default value is 0.5.
    @param model - Optional parameter that specifies the model to be used. Default value is 'gpt-3.5-turbo'.
    @param apiKey - The API key for authorization.
    @returns An Observable that emits string values representing the generated text updates from the API.
*/
  public doOpenAICall(
    messages: Message[],
    temperature: number = 0.5,
    model: string = 'gpt-3.5-turbo',
    apiKey: string
  ): Observable<string> {
    const url = 'https://api.openai.com/v1/chat/completions';

    return new Observable((observer) => {
      const xhr = new XMLHttpRequest();
      xhr.open('POST', url);
      xhr.setRequestHeader('Content-Type', 'application/json');
      xhr.setRequestHeader('Authorization', 'Bearer ' + apiKey);

      // Callback function that gets called as the response progresses
      xhr.onprogress = () => {
        // Extract new updates from the response text
        const newUpdates = xhr.responseText
          .replace('data: [DONE]', '')
          .trim()
          .split('data: ')
          .filter(Boolean);

        // Parse the new updates and extract the generated text
        const newUpdatesParsed: string[] = newUpdates.map((update) => {
          const parsed = JSON.parse(update);
          return parsed.choices[0].delta?.content || '';
        });

        // Emit the generated text updates to the observer
        observer.next(newUpdatesParsed.join(''));
      };

      xhr.onreadystatechange = () => {
        // 4 = DONE
        if (xhr.readyState === 4) {
          if (xhr.status === 200) {
            // Signal completion to the observer
            observer.complete();
          } else {
            // Signal error to the observer
            observer.error(
              new Error('Request failed with status ' + xhr.status)
            );
          }
        }
      };

      // Send the request to the OpenAI API
      xhr.send(
        JSON.stringify({
          model,
          messages,
          temperature,
          frequency_penalty: 0,
          presence_penalty: 0,
          stream: true, // set this to false if you don't want to have streaming
        })
      );

      // Return a cleanup function that aborts the request if the observer unsubscribes
      return () => {
        xhr.abort();
      };
    });
  }
}
```

## Calling the `doOpenAICall` method
In our component, we created an input for our OpenAI API key and a button to call the `doOpenAICall` method. 
When the button is clicked, we call the `doOpenAICall` method with a single message. 

This message is the prompt for the OpenAI API. The OpenAI API will then generate a response to this prompt. 
The response will be emitted to the `openAiResult$` observable. We use the `async` pipe to subscribe to this observable for us, so we don't have to unsubscribe manually.

Feel free to change the prompt or add multiple messages to the `messages` array to see which result you get.

```typescript
@Component({
  selector: 'my-app',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <label for="apikey">Enter you OpenAI API key:</label>
    <input name="apikey" [(ngModel)]="apiKey"> 
    <br>
    <button (click)="doOpenAICall()">Click me for a surprise</button>

    <pre *ngIf="openAiResult$ | async as result">{{result}}</pre>
  `,
})
export class App {
  private readonly openAiService = inject(OpenAIService);
  public apiKey: string = '';
  public openAiResult$ = of('');

  public doOpenAICall() {
    const messages: Message[] = [
      {
        // TODO Change this to your own prompt
        content:
          'Write a small rap song about 2 potatoes that are in love with Angular',
        role: 'user',
      },
    ];

    this.openAiResult$ = this.openAiService.doOpenAICall(
      messages,
      0.5,
      'gpt-3.5-turbo',
      this.apiKey
    );
  }
}
```

## Conclusion
That's all we need to call the OpenAI API from Angular. Feel free to play around with the [StackBlitz](https://stackblitz.com/edit/angular-fqjkmx?file=src/main.ts){:target="blank"} below.
 
<iframe src="https://stackblitz.com/edit/openai-angular" width="100%" height="500px"></iframe>




