# LLM integration in C#

Integration of an LLM in a C# project is pretty simple. Microsoft comes with a NuGet package abstraction, Microsoft.Extensions.AI, that is able to work with any LLM, either cloud-based, on-prem, or locally running.
For the locally running model, you can directly use OllamaSharp, which uses the Ollama runtime on localhost. The OllamaSharp is compatible with Microsoft.Extensions.AI, so you can combine them.

## Ollama

Simply put, the Ollama is an LLM runtime built on top of llama.cpp. It has its own HTTP Server with its API. However, it's compatible with the ChatGPT API, so most SDK that targets ChatGPT can also be used with Ollama.
I've been using edge models `gemma4:e2b`, `gemma4:e4b`, and medium size mode `gemma4:26b`. Unfortunately, the `gemma4:26b` at that time had [_"Double BOS" and Reasoning Desync_](https://gemini.google.com/app/f0d06d389086adb3) bug.

### Integration
I've been working on ASP.NET integration, so I've decided to use Microsoft.Extensions.AI with OllamaSharp. Unfortunately, I've used Microsoft.Extensions.AI only partially, and at the end of the day, it does not make sense the way I did.
I'll point to these odds in the text.

Starting with dependency injection integration.

```csharp
// IChatClient is an abstract interface from Microsoft.Extensions.AI
services.AddSingleton<IChatClient>(_ =>
{
    // OllamaApiClient implementes the IChatClient
    var ollama = new OllamaApiClient(new OllamaApiClient.Configuration
    {
        Uri = new Uri("http://localhost:11434"),
        Model = "gemma4:e4b",
    });

    return ollama;
});
```

For demo purposes, I used the `IChatClient` right in the controller (YOLO!). The first odd thing that you noticed immediately is casting from `IChatClient` to `IOllamaApiClient`. The `OllamaApiClient` supports both interfaces. I haven't revisited the code, but now I would rather use `IChatClient` along the way, or rather register the LLM client as `IOllamaApiClient`. The reason, why I've decided to cast it to the `IOllamaApiClient` is to be able to extend the client with external tools. You can extend `IChatClient` with external tools at registration into the DI. In the case of `IOllamaApiClient`, you extend the client with external tools when you send the prompt.
For prototyping, it's been just simpler to use the approach that `IOllamaApiClient` does.

```csharp
/// <summary>
/// AI chat endpoint (shorten)
/// </summary>
/// <param name="clientId">Required Client ID</param>
/// <param name="request">Chat history</param>
/// <param name="token">HTTP Cancellation token</param>
/// <returns>Appropriate HTTP response</returns>
[HttpPost("clients/{clientId}/ai/chat")]
[ProducesResponseType((int) HttpStatusCode.OK, Type = typeof(AssistantResult))]
public async Task<IActionResult> PostClientResourceSync(
    [FromRoute][Required] ClientId clientId,
    [FromBody] ChatHistoryRequest request,
    CancellationToken token)
{
    var client = (IOllamaApiClient) _chatClient;
    var chat = new Chat(client)
    {
        Think = ThinkValue.Medium,
        Options = new RequestOptions
        {
            // Lower temperature, so the LLM would not be that creative with answers
            Temperature = 0.2f
        }
    };

    // Capture reasoning
    var thinking = new StringBuilder();
    chat.OnThink += (_, thoughts) => thinking.Append(thoughts);

    // Add system prompt that describes what is expected and what the LLM should do
    chat.Messages.Add(new Message(ChatRole.System, SystemPrompt));

    // Define external tools
    IEnumerable<object> tools = [
        // This is the callable function factory, that is able to get services from DI
        new SecurityTools.IsAuthorizedTool(
            HttpContext.RequestServices.GetRequiredService<ISecurityAccessValidator>(),
            HttpContext.RequestServices.GetRequiredService<IEscherUtilityRepository>()),
    ];

    var sb = new StringBuilder();
    // LLM should response in JSON schema, register the schema and send it over to LLM
    var format = JsonElement(ResponseSchema); // This can be generated on the fly
    await foreach (var tkn in chat.SendAsync(request.Prompt, tools, format: format, cancellationToken: token))
    {
        sb.Append(tkn);
    }

    _logger.LogInformation("[🤔]: {Thoughts}", thinking.ToString());
    _logger.LogInformation("[💡]: {Response}", sb.ToString());
    
    // Here should be retry, improve!
    var response = JsonSerializer.Deserialize<AssistantResult>(sb.ToString(), new JsonSerializerOptions
    {
        PropertyNameCaseInsensitive = true,
        Converters = { new JsonStringEnumConverter() }
    });

    return Ok(response);
}
```

### System Prompt

The System prompt is extremely important! In my case, the AI Agent is used for getting the data from the backend database and in the future, it should be able to run operations over some resources.

To describe what the agent should do, I've used multiple approaches from Prompt Framework. The Prompt Framework defines the following topics.

    - Zero-Shot
    - Few-Shot
    - Chain-of-thought
    - React
    - Role-based
    - Tree-of-thought

In my System prompt, I am using Few-Shots (examples), Chain-of-thought (reasoning), React (external tool calls), and Role-based (role definition). The following example is the System prompt that I've ended up using.

```
You are an expert in natural language search for the Paylocity Position Management API. To get the data based on user input you create a plain filter, no JSON is structure is expected.

The filter supports the following operations:
    - "eq": Equal - example: "firstName eq 'John'"
    - "ne": Not - example: "equal firstName ne 'John'"
    - "gt": Greater than - example: "hourlyWage gt 25"
    - "ge": Greater than or equal - example: "hourlyWage ge 25"
    - "lt": Less than - example: "hourlyWage lt 25"
    - "le": Less than or equal - example: "hourlyWage le 50"
    - "in": Equals any of the values provided	- example: "firstName in ('John', 'Jack')"
    - "ct": Contains the values provided (use carefully and watch performance) - example: "title ct 'Engineer'"
    - "sw": Starts with the value provided (use carefully and watch performance) - example: "salutation sw 'Honour'"
    - "cv": Array attribute contains an element whose value equals the value provided. (only compatible with arrays of strings or primitive types) - example: "aliases cv 'OJ'"

Logical AND or OR can be used within an individual filter, and parenthesis can be used to denote precedence in filters.
Here are a few examples:
    - "age gt 10 AND age lt 60": An age has to be greater than 10 and less than 60
    - "age lt 10 OR age gt 60": An age has to be less than 10 or greater than 60
    - "(age gt 60 OR age lt 10) AND status eq Active": An age has to be greater than 60 or less than 10, and status has to be Active

For positions you are able to create a filter for the following properties:
    - PositionKey, type of int; support "eq" filter operation
    - ClientId, type of int; supports "eq" filter operation
    - Code, type of string; supports "eq", "in", "ct", and "sw" filter operations
    - Title, type of string; supports "eq", "in", "ct", and "sw" filter operations
    - Active, type of bool; supports "eq" filter operation
    - IsDraft, type of bool; supports "eq" filter operation
    - EffectiveDate, type of DateTime; supports "eq", "gt", "ge", "lt", and "le" filter operations
    
No other properties are supported for filter creation. If user ask for filter on unsupported property, return null in PositionFilter field and explain in Message that filter for this property is not supported.
When you receive a message, do the following task in the EXACT order:

    1. If you are asked about topic that are unrelated to the position management, positions, eeo classes, workers compensation codes, pay grades, or position families, you politely reject operation and stop with follow up steps.
    2. Get the Company ID from message. The Company ID is company identifier that can be make up 9 alphanumeric characters, but usually it is shorter than 9 characters. If Company ID is not provided, use 'null' value as Company ID. If you find more than one Company ID immediately return Message: "Only a single Company ID in request is supported" and stop. You're expected to set CompanyId field of result object.
    3. Run authorization function with Client ID '{clientId.Id}' and Company ID that you've identified in the previous step. If the authorization function returns false, then immediately return the message "Access denied to the company data" and stop processing the request and stop. Always run the authorization function even if user don't ask for it, except you receive unreleate question as is mentioned in first step.
    4. If you are able to create position filter, put it into the PositionFilter field in the final JSON response. Always return PositionFilter even if you are not able to create position filter based on user request, in such case return null. If user request is not related to positions, return null in PositionFilter field.

Don't report ongoing progress, only the steps that you've done. Only the final answer is in raw JSON form, you can use any format to be able to call functions. 
In the final response, do not use markdown formatting and do not include ```json tags. Output the final response only the minified JSON string and strictly follow the JSON schema provided in the request format.

Response JSON Schema is following:

\```json
{ResponseSchema}
\```
```

At the beginning of the System prompt, I define LLM's role and what it's used for, followed by instructions on how to generate query filters (query generation). It's a good idea to give a list of examples. Usually, it leads to fewer hallucinations. It's also important to instruct it to work only on a given set of fields that are actually in the app, so it would not hallucinate.
I've also considered generating raw SQL queries, but giving it direct access to the database __is dangerous__. Using Abstract Syntax Tree (AST) for queries is way safer, and I can check what properties are actually used in the query and even do the optimisations.

The following instructions are about the agent's behaviour. Mainly, what to do when it's been asked about an unrelated topic (prompt injection), followed by instructions on how what has to be done (the function call). Finally, it instructs the agent to respond in the JSON format. Unfortunately, the Gemma4 is not able to get the schema from the format parameter, so the schema must be noted in the system prompt.

> **The system call in my example is only a representative example. In the production environment, never ever use an agent to do authorization. Authorization should always be part of your code!**

For the sake of completeness, here is the JSON Schema.

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://paylocity.com/schemas/positionmanagementapi/AssistantResult.schema.json",
    "title": "AssistantResult",
    "type": "object",
    "additionalProperties": false,
    "properties": {
        "message": {
            "type": "string"
        },
        "companyId": {
            "type": ["string", "null"]
        },
        "positionFilter": {
            "type": ["string", "null"]
        }
    },
    "required": ["message"]
}
```

## AI Agent architecture
The first key decision that I had to make was about the AI Agent architecture. The vision was to create an AI Agent that is able to retrieve data based on natural language and even run selected operations on existing positions and position-related resources. Due to limited time, I've limited the scope to only natural language search. I've learned that in the AI Agent architecture, there are a few main approaches:

 - Query generation
 - Agentinc approach
 - Hybrid approach

### Query Generation
In this approach, the AI agent only generates a filter query, and the query is either sent back for execution or it can be executed in the following steps in the algorithm. The AI Agent has to learn how to build the query. With this approach, we have to guarantee that AI will only use the limited filter fields.
The system should restrict agents to use only a set of filter fields, and the operations are available for each field. However, the check for fields must also be inside the follow-up step that executes the query.
The AI Agents sometimes hallucinate and may create fields that aren't supported, so it's important to do the validation. The system can even repair itself, and if the system detects an unknown field, an agent can be asked to use only available fields.

### Agentic Approach
AI Agents can execute functions. The important thing is that they are able to receive the necessary parameters from the context window and use them correctly for the function call. I've experienced a couple of *drawbacks* with this approach.

 - Small models often 
    - Either forget to call the function and then hallucinate the response, or
    - Ignores the instruction on purpose and hallucinates the response call
 - I've also experienced the importance of clear instructions, especially with a big system prompt
    - If I've instructed the agent to return only JSON, it starts calling the external functions with JSON-encoded arguments, and then the agent is not able to find the function
 - Access to the Dependency Injection *is possible*, but...
    - It requires a factory function that has access to the DI
    - The factory returns the *callable function* and all dependencies must be passed as arguments
    - Here is the example of an external function definition for the AI agent's `new SecurityTools.IsAuthorizedTool(_securityAccessValidator, _escherUtilityRepository)`
    - An example of a callable function factory is below

```csharp
public static class SecurityTools
{
    // 1. This is the callable function - required dependencies are passed as arguments
    public static async Task<bool> IsAuthorized(
        ISecurityAccessValidator accessValidator, 
        IEscherUtilityRepository escherUtilityRepository, 
        int clientId, 
        string? companyId)
    {
        return true;
    }
    
    // 2. This is callable function factory - the required dependencies are passed as rguments and forwarded during invoke
    public class IsAuthorizedTool : Tool, IAsyncInvokableTool
    {
        // The dependencies that are use while function is invoked
        private readonly ISecurityAccessValidator _accessValidator;
        private readonly IEscherUtilityRepository _escherUtilityRepository;

        public IsAuthorizedTool(ISecurityAccessValidator accessValidator, IEscherUtilityRepository escherUtilityRepository)
        {
            _accessValidator = accessValidator;
            _escherUtilityRepository = escherUtilityRepository;

            // OllamaShart function schema definition
            Function = new Function
            {
                Name = "IsAuthorized",
                Description =
                        "Authorization function that checks authorization for the client and company. The function return boolean value. Returns 'true' if user is authorized to read data from company in the given client, otherwise 'false'. The Company ID can be null, but the function still has to be called!", 
                Parameters = new Parameters
                {
                    Properties = new Dictionary<string, Property>
                    {
                        { "clientId", new Property { Type = "number", Description = "The client identifier, also known as Client ID" } },
                        { "companyId", new Property { Type = "string", Description = "A nullable company identifier, also known as Company ID" } }
                    },
                    Required = ["clientId", "companyId"]
                }
            };
            Type = "function";
        }
        
        // Async function invocation
        public async Task<object?> InvokeMethodAsync(IDictionary<string, object?>? args)
        {
            // You have to get parameters from the argument dictionary
            args ??= new Dictionary<string, object?>();
            var clientId = Convert.ToInt32(args["clientId"]);
            var companyId = (string?) args["companyId"];

            // 3. Create callable function with dependencies from the 1. step
            return await IsAuthorized(_accessValidator, _escherUtilityRepository, clientId, companyId);
        }
    }
}
```

As mentioned before, I've often seen that the function is not called, and the LLM model hallucinates the function call instead of calling the function. The medium-sized model is better at following instructions. However, I believe even the instructions themselfs has significiant impact.

### Hybrid approah
This is the approach that uses both approaches mentioned above. This is also the approach that I've decided to take. My interest was to develop an AI Agent that is able to execute some operations, so I've intentionally done some part with Query Generation and Authorization check as a function execution. Let's stress more that doing authorization inside the AI model is not something you want to do in production, because you __can not__ rely on the LLM model that it'll perform the function as you expect. I've decided to do it this way simply to get experience with function calls.

### Function invocation
How the function invocation works is quite interesting from my perspective, so let me note it here. The Ollama has an HTTP server, and the SDK sends an HTTP request to it. The Ollama in the HTTP response returns JSON, and it may contain `tool_calls`, and if so, the SDK, via the external tool definition, tries to find a registered function. If it's found, then via reflection it's been invoked, and a response is sent to the Ollama, and the LLM get the data from the call and the previous context and continues to get the expected answer. Pretty simple.

## Required Agent Capability
I've decided that my approach is about defining all tasks that the AI agent must do based on the user prompt. The model will try to follow all instructions inside the __system prompt__ and return __JSON object__ that will contain all answers.
It turned out that this approach is very difficult for small models (2B or 4B) and even a bit difficult with medium models, due to the reasoning. The medium-sized model with reasoning can handle this approach. However, the result is *questionable*. 
The responses contain something that looks like a control parameters comming from the neural network itself. It's interesting because the reasoning contains the correct and expected output.
Here is an example of a *broken* response with reasoning.
```json
{
  "message": "I am sorry, but I cannot assist with that question. I can only assist with questions related to position management, positions, EEO classes, workers compensation codes, and pay grades. Please follow up with the steps provided in the documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation documentation
```

Here is the reasoning behind.

```
    *   User input: "Why is the sky blue?"
    *   Topic: Natural science/meteorology.
    *   Context: Paylocity Position Management API.
    *   Allowed topics: position management, positions, eeo classes, workers compensation codes, pay grades, or position families.
    *   Step 1: Check if the topic is related to the allowed topics.
    *   "Why is the sky blue?" is unrelated to position management, positions, eeo classes, workers compensation codes, pay grades, or position families.
    *   Action: Politely reject the operation and stop.

    *   Message: "I am sorry, but I can only assist with queries related to position management, positions, EEO classes, workers' compensation codes, pay grades, or position families."
    *   CompanyId: null (not provided).
    *   PositionFilter: null.

    *   The instructions say: "If you are asked about topic that are unrelated to the position management... you politely reject operation and stop with follow up steps."
    *   The final response must be a minified JSON string following the schema.

    *   `message`: "I am sorry, but I can only assist with queries related to position management, positions, EEO classes, workers' compensation codes, pay grades, or position families."
    *   `companyId`: null
    *   `positionFilter`: null

    *   No markdown.
    *   No ```json tags.
    *   Minified JSON.

    *   `{"message":"I am sorry, but I can only assist with queries related to position management, positions, EEO classes, workers' compensation codes, pay grades, or position families.","companyId":null,"positionFilter":null}`
```

**Note from future myself:** The observed behaviour is the bug between Ollama and Gemma4:26B, as mentioned at the beginning.

## Dev retro

### A lot of responsibilities in one prompt
I have **unintetually** made a decision to have a lot of responsibilities for an AI agent. That turned out to be very difficult to update the System Prompt. Every change in the system prompt, I had to run full prompt tests, including application build. But mainly, the System prompt change causes the LLM to sometimes take instructions a bit differently, so with every change, you have to run the prompt tests.
In the next project, I would rather use a different approach. I would rather have _more agents_ and each focusing on a different aspect, like checking the prompt context (what the user asks for), getting company ID from context, creating a query, running the function, etc. These would probably be called in the pipeline. I believe this approach is also better suited for small models, similar to one that I've used.

### Testing
The testing was difficult. The app had to be rebuilt every time, so every change in the system prompt required a rebuild. To that, there are two possible solutions.

 1. Create a custom LLM model using `modelfile` and bake the system prompt in. This is a more production-ready solution, so you have a model that already has a system prompt. You don't need to attach it from code. However, every chat or request will be served with that system prompt. Use it cautiously.
 2. Put the System prompt into `ApplicationSettings.{env}.json`, which allows dynamic reload for the system prompt.

### Explore different LLM models
Each LLM model is more or less different. The Gemma models are trained to be more conversation-oriented. On the other hand, Phi LLM models are more engineering-oriented, so using different LLM models requires a slightly different approach and adjusting the System Prompts.