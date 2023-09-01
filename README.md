# Amazon Bedrock을 이용하여 Conversational Chatbot 만들기

여기서는 Amazon Bedrock의 LLM(Large language Model)을 이용하여 Prompt에 기반한 conversational chatbot을 구현합니다. chatbot으로 메시지를 전송하면, LLM을 통해 답변을 얻고 이를 화면에 보여줍니다. 입력한 모든 내용은 DynamoDB에 call log로 저장됩니다. 또한 파일 버튼을 선택하여, TXT, PDF, CSV와 같은 문서 파일을 Amazon S3로 업로드하고, 텍스트를 추출하여 문서 요약(Summerization) 기능을 사용할 수 있습니다.

LLM 어플리케이션 개발을 위해 LangChain을 활용하였으며, Bedrock이 제공하는 LLM 모델을 확인하고, 필요시 변경할 수 있습니다. Chatbot API를 테스트 하기 위하여 Web Client를 제공합니다. AWS CDK를 이용하여 chatbot을 위한 인프라를 설치하면, ouput 화면에서 브라우저로 접속할 수 있는 URL을 알수 있습니다. Bedrock은 아직 Preview로 제공되므로, AWS를 통해 Preview Access 권한을 획득하여야 사용할 수 있습니다.

<img src="https://github.com/kyopark2014/simple-chatbot-using-LLM-based-on-amazon-bedrock/assets/52392004/a62d871e-ad88-400b-9d80-6cdf8b3d63a7" width="800">

상세한 동작시나리오는 [Call Flow](https://github.com/kyopark2014/conversational-chatbot/blob/main/call-flow.md)을 참조합니다.


## LangChain 

아래와 같이 model id와 Bedrock client를 이용하여 LangChain을 정의합니다.

```python
from langchain.llms.bedrock import Bedrock

modelId = 'amazon.titan-tg1-large'  # anthropic.claude-v1
parameters = {
    "maxTokenCount":512,
    "stopSequences":[],
    "temperature":0,
    "topP":0.9
}

llm = Bedrock(model_id=modelId, client=boto3_bedrock, model_kwargs=parameters)
```


아래와 같이 Conversation을 설정합니다.

```python
from langchain.chains import ConversationChain
from langchain.memory import ConversationBufferMemory
memory = ConversationBufferMemory()
conversation = ConversationChain(
    llm=llm, verbose=True, memory=memory
)
```


## 대화하기

## 질문/답변하기 (Prompt)

### LangChain을 이용한 기본 대화

LangChang을 대화를 할 수 있습니다.

```python
msg = conversation.predict(input=text)
```

### Prompt Template에 History를 포함하는 방식

```python
history = []
msg = get_answer_using_template(text)

def get_answer_using_template(query):    
    prompt_template = """Human: Use the following pieces of context to provide a concise answer to the question at the end. If you don't know the answer, just say that you don't know, don't try to make up an answer.

    Current conversation:
    {history}

    Human: {input}
    Assistant:"""
    PROMPT = PromptTemplate(
        template=prompt_template, input_variables=["history", "input"])

    #print('result: ', result)
    from langchain.chains import LLMChain
    chain = LLMChain(llm=llm, prompt=PROMPT)

    result = chain.run({
        'history': history,
        'input': query})
    
    history.append('Human: '+query)
    history.append('Assistant: '+result)
    
    return result
```


## 문서 요약하기 (Summerization)

### 파일 읽어오기

S3에서 아래와 같이 Object를 읽어옵니다.

```python
s3r = boto3.resource("s3")
doc = s3r.Object(s3_bucket, s3_prefix + '/' + s3_file_name)
```

pdf파일은 PyPDF2를 이용하여 S3에서 직접 읽어옵니다.

```python
import PyPDF2

contents = doc.get()['Body'].read()
reader = PyPDF2.PdfReader(BytesIO(contents))

raw_text = []
for page in reader.pages:
    raw_text.append(page.extract_text())
contents = '\n'.join(raw_text)    
```

파일 확장자가 txt이라면 body에서 추출하여 사용합니다.
```python
contents = doc.get()['Body'].read()
```

파일 확장자가 csv일 경우에 CSVLoader을 이용하여 읽어옵니다.

```python
from langchain.document_loaders import CSVLoader
body = doc.get()['Body'].read()
reader = csv.reader(body)
contents = CSVLoader(reader)
```

### 텍스트 나누기 

문서가 긴 경우에 token 크기를 고려하여 아래와 같이 chunk들로 분리합니다. 이후 Document를 이용하여 앞에 3개의 chunk를 문서로 만듧니다.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.docstore.document import Document

text_splitter = RecursiveCharacterTextSplitter(chunk_size = 1000, chunk_overlap = 0)
texts = text_splitter.split_text(new_contents)
print('texts[0]: ', texts[0])

docs = [
    Document(
        page_content = t
    ) for t in texts[: 3]
]
```

### Template를 이용하여 요약하기

Template를 정의하고 [load_summarize_chain](https://sj-langchain.readthedocs.io/en/latest/chains/langchain.chains.summarize.__init__.load_summarize_chain.html?highlight=load_summarize_chain)을 이용하여 summarization를 수행합니다.

```python
from langchain import PromptTemplate
from langchain.chains.summarize import load_summarize_chain

prompt_template = """Write a concise summary of the following:

{ text }
        
    CONCISE SUMMARY """

PROMPT = PromptTemplate(template = prompt_template, input_variables = ["text"])
chain = load_summarize_chain(llm, chain_type = "stuff", prompt = PROMPT)
summary = chain.run(docs)
print('summary: ', summary)

if summary == '':  # error notification
    summary = 'Fail to summarize the document. Try agan...'
    return summary
else:
    return summary
```

## LLM 모델 변경

[모델 변경](https://github.com/kyopark2014/conversational-chatbot/blob/main/model-change.md)의 명령어로 채팅창에서 전환할 수 있는 모델의 이름들을 찾고, 특정 모델로 변경할 수 있습니다.



## IAM Role

Bedrock의 IAM Policy는 아래와 같습니다.

```java
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "bedrock:*"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "BedrockFullAccess"
        }
    ]
}
```

이때의 Trust relationship은 아래와 같습니다.

```java
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "sagemaker.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "bedrock.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Lambda가 Bedrock에 대한 Role을 가지도록 아래와 같이 CDK에서 IAM Role을 생성할 수 있습니다.

```python
const roleLambda = new iam.Role(this, "api-role-lambda-chat", {
    roleName: "api-role-lambda-chat-for-bedrock",
    assumedBy: new iam.CompositePrincipal(
        new iam.ServicePrincipal("lambda.amazonaws.com"),
        new iam.ServicePrincipal("sagemaker.amazonaws.com"),
        new iam.ServicePrincipal("bedrock.amazonaws.com")
    )
});
roleLambda.addManagedPolicy({
    managedPolicyArn: 'arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole',
});

const SageMakerPolicy = new iam.PolicyStatement({  // policy statement for sagemaker
    actions: ['sagemaker:*'],
    resources: ['*'],
});
const BedrockPolicy = new iam.PolicyStatement({  // policy statement for sagemaker
    actions: ['bedrock:*'],
    resources: ['*'],
});
roleLambda.attachInlinePolicy( // add sagemaker policy
    new iam.Policy(this, 'sagemaker-policy-lambda-chat-bedrock', {
        statements: [SageMakerPolicy],
    }),
);
roleLambda.attachInlinePolicy( // add bedrock policy
    new iam.Policy(this, 'bedrock-policy-lambda-chat-bedrock', {
        statements: [BedrockPolicy],
    }),
);    
```


## 실습하기

### CDK를 이용한 인프라 설치

[인프라 설치](https://github.com/kyopark2014/chatbot-based-on-bedrock-anthropic/blob/main/deployment.md)에 따라 CDK로 인프라 설치를 진행합니다. [CDK 구현 코드](./cdk-bedrock-simple-chatbot/README.md)에서는 Typescript로 인프라를 정의하는 방법에 대해 상세히 설명하고 있습니다.


### 실행결과

아래와 같이 Converstion이 적용된 동작을 확인할 수 있습니다.

아래 결과는 첫번째에 실패후 두번째에 성공하고 있습니다. 새로운 케이스로 다시 테스트 예정입니다.

![noname](https://github.com/kyopark2014/conversational-chatbot/assets/52392004/2e6c9f30-ca69-4a7f-a981-4895a292a449)


## Debugging

아래와 같이 [test.py](./lambda-chat/test.py)를 이용하여 local에서 컨테이너 이미지를 디버깅할 수 있습니다. 먼저 아래와 같이 이미지를 빌드합니다.

```text
docker build -t lambda_function-test:v1 .
```

Docker를 실행합니다.
```text
docker run -d -p 8080:8080 lambda_function-test:v1
```

아래와 같이 "docker ps"명령어로 Container ID를 확인 할 수 있습니다.
```text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                    NAMES
41e297948511   inference:v1   "/lambda-entrypoint.…"   6 seconds ago   Up 4 seconds   0.0.0.0:8080->8080/tcp   stupefied_carson
```

아래와 같이 Bash shell로 접속합니다.
```text
docker exec -it  41e297948511 /bin/bash
```

Container 접속 후 아래 명령어로 동작을 확인합니다.

```text
cd .. && python3 test.py
```

## Reference 

[How to customize conversational memory](https://python.langchain.com/docs/modules/memory/conversational_customization)

[langchain.chains.conversation.base.ConversationChain](https://api.python.langchain.com/en/latest/chains/langchain.chains.conversation.base.ConversationChain.html?highlight=conversationchain#langchain.chains.conversation.base.ConversationChain)

[langchain.chains.conversational_retrieval.base.ConversationalRetrievalChain](https://api.python.langchain.com/en/latest/chains/langchain.chains.conversational_retrieval.base.ConversationalRetrievalChain.html)

[Store and reference chat history](https://python.langchain.com/docs/use_cases/question_answering/how_to/chat_vector_db)

[Tutorial: ChatGPT Over Your Data](https://blog.langchain.dev/tutorial-chatgpt-over-your-data/)

[LLM memory abstractions](https://www.linkedin.com/pulse/llm-memory-abstractions-maksud-ibrahimov/)


[ValidationError: 1 validation error for ConversationalRetrievalChain](https://github.com/langchain-ai/langchain/issues/6635)

[ConversationalRetrievalChain + Memory](https://github.com/langchain-ai/langchain/issues/2303)
