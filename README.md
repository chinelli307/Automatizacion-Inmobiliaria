# Real Estate WhatsApp Assistant

WhatsApp assistant for real estate agencies, built with n8n and OpenAI. It answers questions about the agency's flats and houses, asks the client what they are looking for, sends photos and passes visit requests to the agency by email. It works with text, voice notes and images.

I built it as a personal project between October 2025 and April 2026. I tested it with a base of 100 Barcelona listings. It is not used by a real agency yet.

## How it works

The workflow has four parts.

### 1. Input

![Input](docs/1-input.png)

Evolution API sends every WhatsApp message to an n8n webhook. Only messages from the client go through. Then a switch checks the type:

- Text goes straight to the next step.
- Voice notes are downloaded, converted to MP3 and transcribed with OpenAI.
- Images are described with GPT-4o-mini so the agent knows what the client sent.

### 2. Waiting for the client to finish writing

![Buffer](docs/2-buffer.png)

On WhatsApp people usually send several short messages in a row ("hi", "I'm looking for a flat", "in Gràcia"). I didn't want the bot to answer each one separately.

Each message is saved in a Redis list with the phone number as the key. The workflow waits a moment and reads the list again. If the last message in the list is still the one this execution saved, the client has stopped writing, so it takes all the messages, deletes the list and sends them to the agent together. If a newer message arrived, this execution stops and the newer one takes care of it.

### 3. AI agent

![Agent](docs/3-agent.png)

The agent uses GPT-4.1 and has these tools:

- `informacion_casas`: searches the listings in Airtable. The agent has to use it before saying anything about a property, so it doesn't make up prices or features.
- `enviar_imagen`: calls a second workflow (`Fotos Inmobiliaria`) with the property ID, for example `BCN-0001`. That workflow finds the property's folder in Google Drive, downloads the photos and sends them to the client's chat.
- `enviar_email`: sends the agency the client's name, phone, preferred date and the property they want to visit.
- Think: a last check before answering, so the reply doesn't invent data and answers what the client asked.

The conversation history is saved in PostgreSQL by phone number, so the agent remembers the client if they write again later.

Before showing any property, the agent asks what the client wants: rent or buy, area, budget and number of rooms. Then it only shows the ones that match.

### 4. Sending the answer

![Output](docs/4-output.png)

A long answer in one message doesn't look natural on WhatsApp. A second model (temperature 0) splits the answer into up to 5 short messages and returns them as JSON. Each message is sent with a small delay depending on its length, as if someone were typing.

## Tools

n8n, OpenAI (GPT-4.1, GPT-4o-mini and speech to text), Evolution API, Redis, PostgreSQL, Airtable, Google Drive and SMTP.

## Limitations

- The Airtable search reads the whole table. With 100 listings it's fine, but with thousands it would be better to filter by area, price and rooms.
- For images it uses the WhatsApp thumbnail, not the full-size picture.
- If the client sends the same text twice in a row (for example "ok" and "ok"), both executions can think they are the last one.
- There is no way yet for a person from the agency to take over the conversation.

## Setup

1. Import `Automatización Inmobiliaria.json` and `Fotos Inmobiliaria.json` into n8n.
2. Add the credentials for OpenAI, Evolution API, Redis, PostgreSQL, Airtable, Google Drive and SMTP.
3. Set the webhook of your Evolution API instance to the webhook URL of the main workflow.
4. Add the listings to Airtable, one row per property, with an `ID` column (`BCN-0001`, `BCN-0002`...).
5. In Google Drive, create a folder for each property named with its ID and put its photos inside. Select the parent folder in the `Buscar carpeta del piso` node.
6. In the main workflow, connect the `enviar_imagen` tool to the `Fotos Inmobiliaria` workflow.
