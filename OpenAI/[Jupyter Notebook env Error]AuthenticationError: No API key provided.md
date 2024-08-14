# 문제
AuthenticationError: No API key provided. You can set your API key in code using 'openai.api_key = <API-KEY>', or you can set the environment variable OPENAI_API_KEY=<API-KEY>). 
If your API key is stored in a file, you can point the openai module at it with 'openai.api_key_path = <PATH>'. You can generate API keys in the OpenAI web interface. 
See https://platform.openai.com/account/api-keys for details.

# 해결 방법 찾기
```
import openai
import os

from dotenv import load_dotenv, find_dotenv
_ = load_dotenv(find_dotenv())

openai.api_key  = os.getenv('OPENAI_API_KEY')
```
Andrew 선생님의 ChatGPT Prompt Engineering 강의를 들으면 해당 코드가 있습니다.
Environment Variable을 설정해줘야하는 문제인데 보통 설정할 때 .env파일을 만들어서 사용하지만 Jupyter Notebook에서도 똑같이 사용하면 되는지 의문이 들었습니다!
그래서 해결 방법을 찾아보던 중 os 라이브러리에서 추가할 수 있는 방법이 있다고 해서 사용해봤습니다!

# 해결
```
import os

os.environ['OPENAI_API_KEY'] = 'API-KEY'

api_key = os.environ['OPENAI_API_KEY']
```
위 코드를 최 상단 코드에 추가해주면 됩니다!
현재 세션에서만 적용되기 때문에 참고하세요!
