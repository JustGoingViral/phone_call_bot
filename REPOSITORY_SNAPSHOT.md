# Repository Snapshot - phone_call_bot

**Generated on:** 2026-01-05

This document contains a complete snapshot of all files in the phone_call_bot repository.

---

## Table of Contents

1. [README.md](#readmemd)
2. [main.py](#mainpy)
3. [models.py](#modelspy)
4. [requirements.txt](#requirementstxt)
5. [.gitignore](#gitignore)
6. [prompts/vet_prompt.md](#promptsvet_promptmd)
7. [prompts/dr_prompt.md](#promptsdr_promptmd)
8. [prompts/customer_support.md](#promptscustomer_supportmd)

---

## README.md

**File Path:** `/README.md`

```markdown
# Voice Call with ChatGPT

This repository is a demo of what'd be possible if you could have ChatGPT to place a call on your behalf. This is **experimental** and not meant to actually be used for placing calls. But it's pretty fun to play around with.

Originally, it was built using the [Eleven Labs](https://beta.elevenlabs.io/) Voice AI API so that you could create your own voice and then have it act as the voice of ChatGPT. But it's been updated to use OpenAI's voice API instead. You could always swap these out if you wanted.

## How to use

Install the dependencies:

```
pip install -r requirements.txt
```

I've included some example prompts for common scenarios where you might want a bot to place a call for you. Of course, you could also add your own.

To run a scenario, run the following command:

```
python main.py -pf 'path/to/prompt/file.txt'
```

Replacing the path above with the path to the prompt file you want to use.

When it starts, you'll see a "Listening..." message. That's essentially the same as a phone call being picked up. You can then play the role of the receiver, ChatGPT will play the role of the caller.
```

---

## main.py

**File Path:** `/main.py`

```python
import os
import datetime
from openai import OpenAI
from elevenlabs.client import ElevenLabs, Voice
from elevenlabs import stream
import argparse
from dataclasses import asdict
from models import Message
import speech_recognition as sr
import logging
from pathlib import Path
logging.basicConfig(level=logging.INFO)
import dotenv
dotenv.load_dotenv('.env')

oai_client = OpenAI()
elevenlabs_client = ElevenLabs()

CHAT_MODEL = "gpt-4o"
TTS_MODEL = "tts-1"
MODEL_TEMPERATURE = 0.5
AUDIO_MODEL = "whisper-1"
VOICE_ID = os.getenv("ELEVENLABS_VOICE_ID")

def ask_gpt_chat(prompt: str, messages: list[Message]):
    """Returns ChatGPT's response to the given prompt."""
    system_message = [{"role": "system", "content": prompt}]
    message_dicts = [asdict(message) for message in messages]
    conversation_messages = system_message + message_dicts
    response = oai_client.chat.completions.create(
        model=CHAT_MODEL,
        messages=conversation_messages,
        temperature=MODEL_TEMPERATURE
    )
    return response.choices[0].message.content

def setup_prompt(prompt_file: str = 'prompts/vet_prompt.md') -> str:
    """Creates a prompt for gpt for generating a response."""
    with open(prompt_file) as f:
        prompt = f.read()

    return prompt

def get_transcription(file_path: str):
    audio_file= open(file_path, "rb")
    transcription = oai_client.audio.transcriptions.create(
        model=AUDIO_MODEL, 
        file=audio_file
    )
    return transcription.text

def record():
    # load the speech recognizer with CLI settings
    r = sr.Recognizer()

    # record audio stream from multiple sources
    m = sr.Microphone()

    with m as source:
        r.adjust_for_ambient_noise(source)
        logging.info(f'Listening...')
        audio = r.listen(source)

    # write audio to a WAV file
    timestamp = datetime.datetime.now().timestamp()
    with open(f"./recordings/{timestamp}.wav", "wb") as f:
        f.write(audio.get_wav_data())
    transcript = get_transcription(f"./recordings/{timestamp}.wav")
    with open(f"./transcripts/{timestamp}.txt", "w") as f:
        f.write(transcript)
    return transcript

def oai_text_to_speech(text: str):
    timestamp = datetime.datetime.now().timestamp()
    speech_file_path = Path(__file__).parent / f"outputs/{timestamp}.mp3"
    response = oai_client.audio.speech.create(
        model=TTS_MODEL,
        voice="nova",
        input=text
    )
    response.write_to_file(speech_file_path)
    return speech_file_path

def elevenlabs_text_to_speech(text: str):
    audio_stream = elevenlabs_client.generate(
        text=text,
        voice=Voice(
            voice_id=VOICE_ID
        ),
        stream=True
    )
    stream(audio_stream)

def clean_up():
    logging.info('Exiting...')
    # Delete all the recordings and transcripts
    for file in os.listdir('./recordings'):
        os.remove(f"./recordings/{file}")
    for file in os.listdir('./transcripts'):
        os.remove(f"./transcripts/{file}")
    for file in os.listdir('./outputs'):
        os.remove(f"./outputs/{file}")
    # Save the conversation
    timestamp = datetime.datetime.now().timestamp()
    with open(f'logs/conversation_{timestamp}.txt', 'w') as f:
        for message in conversation_messages:
            f.write(f"{message.role}: {message.content}\n")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("-pf", "--prompt_file", help="Specify the prompt file to use.", type=str)
    parser.add_argument("-tts", "--tts_type", help="Specify the TTS type to use.", type=str, default="openai", choices=["openai", "elevenlabs"])
    args = parser.parse_args()
    prompt_file = args.prompt_file
    tts_type = args.tts_type or "openai"

    prompt = setup_prompt(prompt_file)
    conversation_messages = []
    while True:
        try:
            user_input = record()
            logging.info(f'Receiver: {user_input}')
            conversation_messages.append(Message(role="user", content=user_input))
            answer = ask_gpt_chat(prompt, conversation_messages)
            logging.info(f'Caller: {answer}')
            logging.info('Playing audio...')
            if tts_type == "elevenlabs":
                elevenlabs_text_to_speech(answer)
            else:
                audio_file = oai_text_to_speech(answer)
                # Play the audio file
                os.system(f"afplay {audio_file}")
            conversation_messages.append(Message(role="assistant", content=answer))
            if 'bye' in user_input.lower():
                clean_up()
                break
        except KeyboardInterrupt:
            clean_up()
            break
```

---

## models.py

**File Path:** `/models.py`

```python
from dataclasses import dataclass
from typing import Optional

@dataclass(frozen=True)
class Message:
    role: str
    content: Optional[str] = None

    def render(self):
        result = self.role + ":"
        if self.content is not None:
            result += " " + self.content
        return result
```

---

## requirements.txt

**File Path:** `/requirements.txt`

```text
openai==1.30.3
SpeechRecognition==3.9.0
PyAudio==0.2.13
elevenlabs==1.2.2
```

---

## .gitignore

**File Path:** `/.gitignore`

```text
logs/
outputs/
recordings/
transcripts/
.env
__pycache__/
.mypy_cache/
```

---

## prompts/vet_prompt.md

**File Path:** `/prompts/vet_prompt.md`

```markdown
You're a helpful assistant that has been tasked to schedule a vet appointment for a pet. You have been provided with the following information:

* The pet owner's name is Test User
* The pet owner's phone number is 555-555-5555
* The pet owner's email is hello@example.com
* The pet owner's address is 123 Fake St, New York, NY 10001
* The pet's name is Fido
* The pet is a dog
* The pet is a previous patient of the vet
* The reason for the appointment is because it needs a checkup

The pet owner's availability for an appointment is:
* Any day during the week between 12pm and 2pm

Your objective is to make an appointment with the vet. You shouldn't schedule or say something that is impossible. For example, you shouldn't make an appointment that falls outside of the user's availability.

If you don't know how to respond, you can say "Sorry, I'm not sure."

Begin.
```

---

## prompts/dr_prompt.md

**File Path:** `/prompts/dr_prompt.md`

```markdown
You're a helpful assistant that has been tasked to schedule a doctor's appointment for a patient. You have been provided with the following information:

* The patient's name is Test User
* The patient's phone number is 555-555-5555
* The patient's email is hello@example.com
* The patient's address is 123 Fake St, New York, NY 10001
* The patient is a patient of Dr. Smith
* The reason for the appointment is because the patient needs a checkup

The patient's availability for an appointment is:
* Monday, Tuesday, or Thursday between 12pm and 2pm

Your objective is to make an appointment with the doctor. You shouldn't schedule or say something that is impossible. For example, you shouldn't make an appointment that falls outside of the patient's availability.

If you don't know how to respond, you can say "Sorry, I'm not sure."

Begin.
```

---

## prompts/customer_support.md

**File Path:** `/prompts/customer_support.md`

```markdown
You've been tasked to call an airline's customer support line and reschedule your flight. You have been provided with the following information:

* Your name is Test User
* Your phone number is 555-555-5555
* Your email is hello@example.com
* Your address is 123 Fake St, New York, NY 10001
* Your original flight was United Airlines flight 1234
* Your original flight was scheduled to depart at 12pm on Monday, January 1st.
* You'd like to reschedule your flight to depart on January 3rd.

If you don't know how to respond, you can say "Sorry, I'm not sure."

Begin.
```

---

## Repository Statistics

- **Total Files:** 8
- **Python Files:** 2
- **Markdown Files:** 4
- **Configuration Files:** 2
- **Lines of Code (approx):** ~200

---

*End of Repository Snapshot*
