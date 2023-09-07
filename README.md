# tgi-chat-ui-function-calling
Add function calling to chat-ui with text-generation-inference

## Before you start
1. Function calling is very sensitive to the prompt setup.
1. This code will integrate function calling so that the language model returns a structured request/function call. However, this code does not include handling of that structured request (i.e. sending it to an api and returning a response to the llm).

Other resources:
1. Full guided instructions for installation of tgi and chat-ui on an AWS server are available [here].(https://buy.stripe.com/9AQ28UcWh4PF1ckeV9)
2. 13b and 34b size function calling models are available for purchase [here on HuggingFace](https://huggingface.co/Trelis/Llama-2-13b-chat-hf-function-calling-v2).

## text-generation-inference Setup (on a Ubuntu server)
Create a script to run 'tgi' with:
```
touch tgi.sh
```
Then provide permission to execute the script:
```
chmod +x tgi.sh
```
Then open the script with:
```
nano tgi.sh
```
and copy paste in the following (the token is required if using 13b or 70b [coming soon] gated models from huggingface.co/trelis):
```
model=Trelis/Llama-2-7b-chat-hf-function-calling-v2
token=<your huggingface token for the trelis repo>
volume=$PWD/data # share a volume with the Docker container to avoid downloading weights every run

docker run -d --gpus all --shm-size 1g -e HUGGING_FACE_HUB_TOKEN=$token -p 8080:80 -v $volume:/data ghcr.io/huggingface/text-generation-inference:1.0.2 --model-id $model --quantize bitsandbytes-nf4 --max-input-length 4000 --max-total-tokens 4096
```
Then save and exit with 'Ctrl + x', then 'y', then 'Enter'.

To run tgi, do:
```
./tgi.sh
```
## chat-ui Setup (on a Ubuntu server)
[Github repo](https://github.com/huggingface/chat-ui/)
Start by cloning the repo:
```
git clone https://github.com/huggingface/chat-ui.git
```
Then start a mongo database:
```
docker run -d -p 27017:27017 --name mongo-chatui mongo:latest
```
Then, we need npm, which requires an updated node, which requires nvm to install. First install nvm:
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```
then run the commands shown on the terminal OR sudo reboot. Then
```
nvm install node
```
Then
```
cd chat-ui
```
then
```
npm install
```
Now, we need to configure the model, so:
```
touch .env.local
```
then
```
nano .env.local
```
and paste in (with your huggingface access token, or delete that line if using a public model):
```
MONGODB_URL=mongodb://localhost:27017/
HF_ACCESS_TOKEN=<YOUR TOKEN>
PUBLIC_APP_NAME="Trelis Research Chat"
PUBLIC_APP_ASSETS=chatui
PUBLIC_APP_COLOR=blue

MODELS=`[
{
        "name": "Trelis/Llama-2-7b-chat-hf-function-calling-v2",
        "datasetName": "Trelis/function_calling_extended",
        "description": "function calling Llama-7B-chat",
        "websiteUrl": "https://research.Trelis.com",
        "userMessageToken": "",
        "userMessageEndToken": " [/INST] ",
        "assistantMessageToken": "",
        "assistantMessageEndToken": " </s><s>[INST] ",
        "chatPromptTemplate": "<s><FUNCTIONS>{functionList}</FUNCTIONS>\n\n[INST] {{#each messages}}{{#ifUser}}{{content}} [/INST]\n\n{{/ifUser}}{{#ifAssistant}}{{content}} </s><s>[INST] {{/ifAssistant}}{{/each}}",
        "parameters": {
                "temperature": 0.2,
                "top_p": 0.95,
                "repetition_penalty": 1.2,
                "top_k": 50,
                "truncate": 1024,
                "max_new_tokens": 1024
        },
        "endpoints": [{
                "url": "http://127.0.0.1:8080"
        }]
}
]`
```
Then 'Ctrl + x', 'y', 'Enter' to save and exit.

Next, we need to create a functions.js file in the chat-ui folder with 'touch functions.js' then 'nano functions.js' to open it and paste in and save:
```
export default {
    "function": "search_bing",
    "description": "Search the web for content on Bing. This allows users to search online/the internet/the web for content.",
    "arguments": [
        {
            "name": "query",
            "type": "string",
            "description": "The search query string"
        }
    ]
}
```
The final change needed is to adjust models.ts in the src/lib/server folder. Navigate to that file and open it with 'nano', then paste in the following code just above const modelsRaw = ...
```
import functionList from '../../../functions.js';

console.log("functionList:", functionList);

function escapeSpecialCharacters(value) {
    return value.replace(/[\\]/g, '\\\\')
                .replace(/[\"]/g, '\\"')
                .replace(/[\/]/g, '\\/')
                .replace(/[\b]/g, '\\b')
                .replace(/[\f]/g, '\\f')
                .replace(/[\n]/g, '\\n')
                .replace(/[\r]/g, '\\r')
                .replace(/[\t]/g, '\\t');
}

let stringifiedFunctionList = '';

stringifiedFunctionList += JSON.stringify(functionList, null, 4);

const escapedFunctionList = escapeSpecialCharacters(stringifiedFunctionList);

const modelsString = MODELS.replace('{functionList}', escapedFunctionList);
```
Lastly, scroll down a little more and replace '.parse(JSON.parse(models));' with '.parse(JSON.parse(modelsString));'

You can now add functions (use carriage returns between them) to the functions.js file. What all of this does is stringify the functions so they can be injected into the Model description in .env.local .

Lastly, run chat-ui:
```
npm run dev
```

Now, open up the local host in your browser and start chatting! If you ask for a bing search, chat-ui should return a json object.

Video [here](https://www.loom.com/share/702bc999ced8404a9622fa9309f41e5e?sid=21868076-25a2-4f44-bfc4-0a619d16dcdf)

