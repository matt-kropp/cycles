# Cycles - An agentic work assistant

## Overview

This platform will be the first fully agentic productivity tool that changes the way knowledge workers manage their entire
professional (and possibly personal) life.  This is about rethinking task management in the age of agentic AI.

The core of the platform is a Cycle which is a process loop related to a given objective.  Cycles can be time-based (e.g.
daily, weekly, monthly) and can be topic-based (e.g. relating to Project X or Client Y).  Each Cycle has a Purpose that 
describes what part of the User's life it is managing, a history of Inputs that capture inbound information or events 
(such as emails, Slack emssages, new events, etc.), and a schedule of actions to be taken in the future.  The User can
create new Cycles ad hoc, but mostly Cycles are created when there is new inbound requests for the User's attention.  Each
Cycle keeps a summary of new information gathered since the last information update to the User and keeps track of what
actions the User needs to take, reminding them when actions are coming due.  Cycles can proactively take action on 
behalf of the User of the system has enough information available to take such action.

## Data Model

Cycle - The core entity representing topics or activities that the User needs to manage
 - ID - guid
 - name - a short title for the cycle used in display
 - purpose - a detailed description of the types of topics or activities this Cycle is used to manage; description is used by an AI Agent to determine which Cycle is appropriate for any given action or event
 - purpose_embedding - vector embedding of the Purpose
 - client_id (nullable) - reference to client if applicable
 - topic_id (nullable) - reference to topic if applicable
 - input_history - JSONB array of inbound inputs in form [{type, input_message_id, datetime}] where type is text and is typically "email", "Slack", "user"
 - summary - a comprehensive text summary of the Input_History for this cycle; updated upon each new Input_History entry
 - dailies - an array by day of text summary of the Input_History for that day
 - attention_flag - NONE | ANYTIME | SCHEDULED | URGENT - indicates when User should take action on this Cycle
 - next_action_start_datetime (nullable) - when set represents the datetime by which the User should start to take next action on this Cycle
 - next_action_due_Datetime (nullable) - when set represents the datetime by which the User should complete the next action on this Cycle
 - next_action__description - text description of the action the user needs to take next for this cycle
 - next_actions - JSONB array of all currently planned actions required for the User to take on this cycle - each action has a text description, a start_datetime and a due_datetime
  
Input_Message - input messages associated to Cycles
 - ID - guid
 - external_message_id - ID from external system (e.g. Outlook or Slack) that can be used to link back to the original message in the source system
 - datetime
 - from - sender id in text - can be email or Slack handle or full name depending on source
 - cc - JSONB array of addresses in cc again might be emails or Slack handles etc.
 - subject - text of subject (if any)
 - body - text of message body
 - embedding - vector embedding of subject + body
 - type - IGNORE | INFORMATION | ACTION_REQUIRED - is this message just for information or does it require action or is it zero value and can be ignored?
 - urgency - NONE | ANYTIME | SCHEDULED | URGENT - how urgent is action required on this message?
 - action - text description of the action required (if any)
 - action_start_datetime - when does this action need to be started
 - action_due_datetime - when is this action due
 - client_id (nullable) - reference to Client if applicable (might be derived from the domain in email addresses other than my company domain which is bcg.com)
 - topic_id (nullable) - reference to the general topic of discussion of this message

Chat_Message - chat message history between the User and the system chatbot
 - ID
 - role
 - content
 - datetime
 - cycle_id - (optional) link to a cycle if selected by the  user to referenced in the conversation

Chat_Message_Summary - summary of range of chat messages periodically done to keep context length managable
 - ID
 - starting_chat_message_id
 - ending_chat_message_id
 - summary - text summary of messages between starting and ending id
 - cycles - JSONB array of cycle_ids referenced in chat_messages

Chat_Topics- topics discussed in the User / system chatbot - used to retrieve Chat_Message_Summaries
 - ID
 - chat_message_summary_id - note can be many Chat_Topics to one Chat_Message_Summary
 - topic_summary - short description of the topic covered
 - topic_embedding - vector embedding of the topic_summary

Client - a list of clients referenced in Input_Messages and Cycles
 - ID
 - name
 - domain - email address domain

Topic - a list of topics referenced in Input_Messages and Cycles
 - ID
 - name
 - description - short text description of the topic
 - embedding - vector embedding of the description

## Data flow

An API endpoint is exposed that allows external system to send in new Input_Message events (to start this will be 
Power Automate sending each new Outlook email).  When this API endpoint is called, a new Input_Message is created and 
gpt-5-nano is used to extract type, urgency, action, action_start_datetime, action_due_datetime, client and topic
from the subject, message, from and cc fields and the openai embedding API is used to create the embedding field.  When
prompting the LLM to classify the message, provide a list of currently available clients and topics so that the LLM 
preferentially chooses existing clients or topics but can add a new one if there are no matches.

After a new Input_Message is created, use the client and topic field from the Input_Message to query Cycles and see if there
is an existing Cycle that matches this Input_Message.  If so, associate the two by adding the Cycle.input_history and 
updating dailies, attention_flag, next_action_start_datetime, next_action_due_datetime, next_action_description, and 
next_actions an appropriate.  If no cycle matches, create a new one based on this Input_Message.

## User Inferface

There are three modalities for the User to interact with the system and the UI should provide a well-designed UX that 
enables the user to interact seamlessly with these modalities:

### Cycles UI

The User can see a list of all currently active Cycles which provides key information about the topic / client of the
cycle, whether there is an action urgently due and when the next action is due.  This should be ordered so that the most
urgent Cycles are at the top.  Clicking a Cycle should open a modal with all necessary information about the cycle
visible and should allow the user to take action on the Cycle.  A filter allows the user to filter to only Cycles that 
have new information as of today, so that the User can see what is new incoming information for them to review.  For Cycles
that don't require action, the user to make them as "read" and they then not longer show up in the Cycle view.

### Actions UI

The User can see a list of actions required, ranked by urgency, across all cycles. The User can click on an action and
see the context and be able to take action.

### Chat UI

The User can chat with an agent that has access to all Cycles and can provide answers on actions required.

## Technical Architecture

- Frontend - React / Tanstack / Vite / Typescript / Tailwind / shadcn
- Backend - NodeJS / FastAPI / Typescript
- Database - Postgress / pgvector / DrizzleORM
- Structure the project as a monorepo
- Testing - Vitest
- All LLM calls should be using OpenAI API to gpt-5-nano
- 
