# Building the Chatbot of the Future
### 2026-03-16

Now, I’ll preface this article by reminding you that your app probably doesn’t need a chatbot. Don’t accept that meeting invite to discuss it, don’t add a task to your Jira board, don’t let the idea pass Go and collect $200. Maybe I’m getting old, but the chatbots I see in the wild these days have gotten so useless that sometimes even I just want to call the company and ask my questions, and that’s saying a lot.

However, if you are *convinced* that you need one, then there’s a new library worth playing around with that can help enhance your glorified internal wiki search into a futuristic, UX-enhancing dreambot.

### Enter, @tanstack/ai

[TanStack AI](https://tanstack.com/ai/latest/docs/getting-started/overview) is a framework-agnostic LLM-enabling implementation that abstracts away a whole bunch of setup and state management so you can plug straight in and start your chatbot journey. With libraries like `@tanstack/ai-react` and `@tanstack/ai-preact` (among others), that provide a simple `useChat()` hook that exposes current session’s messages, loading state, new message functions, and more, the hardest decision you’ll need to make is how you want to display the messaging interface to your users. I won’t go into a full-blown example here, there are plenty of these in the documentation, but below is a very short snippet from [this example](https://tanstack.com/ai/latest/docs/api/ai-preact#example-basic-chat).

```jsx
function Chat() {
	const { messages, sendMessage, isLoading } = useChat({
	    connection: fetchServerSentEvents("/api/chat"),
	});

	return <div>...</div>
}
```

And that’s your frontend sorted! Alongside, a simple server implementation (Nodejs example below taken from [here](https://tanstack.com/ai/latest/docs/api/ai#chatoptions)), you’ve got yourself a full-fledged chatbot.

```jsx
import { chat } from "@tanstack/ai";
import { openaiText } from "@tanstack/ai-openai";

const stream = chat({
  adapter: openaiText("gpt-5.2"),
  messages: [{ role: "user", content: "Hello!" }],
  tools: [myTool],
  systemPrompts: ["You are a helpful assistant"],
  agentLoopStrategy: maxIterations(20),
});
```

I don’t know about you, but that’s the easiest chatbot setup I’ve ever seen. Wire this up with a clean system prompt, or your own custom model trained on internal wiki data, and you’re laughing.

But that’s just the start of what you can do in 2026.

### Welcome to the Chatbot Stone Age

The leap that has been taken that, in my opinion, has really started to push chatbot capabilities into the Upper Palaeolithic is the inclusion of the ability to define server and client tools. These tools can be defined to do just about anything, but with the key differentiator being that *you don’t need to identify when to call them*. You simply set up a clearly typed definition of the tool including input and output parameters, define what should happen when the tool is called, and let the LLM of your choice parse the chat request and offload to the right tool with the right data. Not particularly ground-breaking in terms of AI functionality per se, but in terms of web development it’s a game changer.

There’s a few different libraries that I’ve seen starting to build out these tool definitions, but in keeping with the TanStack toolbox (pun intended), these examples will be derived from their documentation + some examples I’ve put together using their libraries. For these examples also, I’ll be using Python API with a Preact frontend - somebody with more time on their hands will surely have created a cleaner Python adapter, so take mine with a grain of salt.

Let’s imagine a very simple example. Say you have a website that has multiple different themes available and you can’t figure out the best CSS animation to use for the component. Well now you don’t have to! All you need to do is set up a tool definition in your API…

```python
theme_entries = [(1,"light"),(2,"dark"),(3,"beige")]
theme_ids = [item_id for item_id, _ in theme_entries]
theme_map_text = ", ".join(f"{item_id}={name}" for item_id, name in theme_entries)

tools = {
    "type": "function",
    "function": {
        "name": "theme",
        "description": "Identify and control the selected app theme",
        "parameters": {
            "type": "object",
            "properties": {
                "theme": {
                    "type": "string",
                    "enum": theme_ids,
                    "description": f"Selected theme ID. Valid IDs: {theme_map_text}",
                },
            },
            "required": ["theme"]
        },
    },
},

...

stream = await client.chat.completions.create(
    model=model,
    messages=messages,
    max_completion_tokens=1024,
    tools=tools,
    tool_choice="auto",
    stream=True,
)
```

then provide a frontend client definition that plugs into your useChat hook…

```tsx
const themeSwitchUIDef = toolDefinition({
  name: 'theme', // must match backend tool definition name
  description: 'Identify and control the selected app theme',
  inputSchema: z
    .object({
      theme: z
        .enum([1,2,3]) // The available theme IDs
        .describe('The selected theme ID'),
    })
  outputSchema: z.object({
    success: z.boolean(), // Not strictly necessary, but I add this out of habit
		// You could make the output type whatever you like, and chain more events afterwards
  }),
})

export const Chat = () => {
    const themeSwitchUITool = themeSwitchUIDef.client(
        (input) => { // The docs say this input should automatically identify the type
            // from the zod schema, but I'm yet to be so lucky
        
        // Add extra validation for safety, but in essence just switch to the chosen theme
        themeSwitcherService.setTheme(input.theme)

        return { success: true }
        },
    )
  
    const tools = clientTools(themeSwitchUITool) // initiate tools from the definitions
  
    const { messages, sendMessage, isLoading } =
        useChat({
            connection: ...,
            tools // pass tools in as a prop
        })
	  
    return <div>...</div>
}
```

and just like that you don’t even need a theme switching button anymore - your users can just ask your chatbot which themes are available and to switch the theme for them!

The keen-eyed among you might notice in this example that there’s some duplicated, hardcoded enum variables for the themes within the tool definitions which is not ideal. So to take it even further, you could dynamically generate your tool definition - possibly from configuration, but could be from a previous API call to retrieve the available themes - and pass the available themes in the payload of your chat requests. And thankfully, the useChat options support just this already.

```tsx
const buildThemeSwitchTool = (themes: Theme[]) => {
  const inputSchema = z
    .object({
      theme: z
        .enum(
          themes.flatMap(theme => theme.id.toString()),
        )
        .optional()
        .describe('The selected theme ID'),
    })

  return toolDefinition({
    name: 'theme',
    description:
      'Identify and control the selected app theme',
    inputSchema,
    outputSchema: z.object({
      success: z.boolean(),
    }),
  })
}

export const Chat = () => {
	// Load dynamically or pass in as props
	const themes = [{id: 1, name: 'Light',{id: 2, name: 'Dark',{id: 3, name: 'Beige',}]
	
	// Create definition dynamically - wrap in a useMemo if you expect data to change
	const themeSwitchUIDef = buildThemeSwitchTool(themes)
  
	const themeSwitchUITool = themeSwitchUIDef.client(
        (input) => {
            themeSwitcherService.setTheme(input.theme)
            return { success: true }
        },
    )
  
    const tools = clientTools(themeSwitchUITool)
  
    const { messages, sendMessage, isLoading } =
        useChat({
            connection: ...,
            body: {
                availableThemes: themes, // pass any extra data in the body
            },
            tools,
        })
	  
    return <div>...</div>
}
```

The above dynamic implementation then mean that, in your API, you can parse out the themes from the `data` object, and put that straight into the same tool definition as before, just without the need to redefine your data.

```python
messages = request.data.get("messages", [])
extra_data = request.data.get("data") or {}
themes = extra_data.get("availableThemes", [])

theme_entries = [
    (str(item["id"]), item.get("name", str(item["id"])))
    for item in backboard_colours
    if isinstance(item, dict) and item.get("id") is not None
]
theme_ids = [item_id for item_id, _ in theme_entries]
theme_map_text = ", ".join(f"{item_id}={name}" for item_id, name in theme_entries)

...
```

This is obviously a bad example - don’t use AI just for your theme switching tool - but it serves the purpose of illustrating how you can extend your app’s chatbot from being purely informative to fully interacting with your app in a way that is still typed and controllable. And with the ability to pass around dynamic data and state alongside the tool definition, you can truly add an extra dimension to the UX of your app. Not only does it enable “text to UI changing action” capability, but could surely be extended out to voice control as well without too much additional work.

### Wrapping up

You might be thinking that there’s still an elephant in the room to address - and I’m right there with you - does this kind of chatbot just shove poor UX decisions under a technologically impressive rug instead of just making your app easier to use? Sure, that could definitely happen if you’re not careful. But if you can still maintain a solid UX foundation throughout, then a chatbot like this should only serve to extend the accessibility of your app. And I don’t know about you, but helping bring some more accessibility into the world of technology seems a much better use of AI then whatever it is Twitter is doing.

I don’t know exactly where things will go from here - I obviously have my guesses - but I’m for sure excited to see what these sorts of features can bring to the web.
(and if anybody *does* use this for their theme switcher tool, let me know)