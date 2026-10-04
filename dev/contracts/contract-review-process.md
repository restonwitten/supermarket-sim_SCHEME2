# Contract Review Process

## Structure

- A <u>Contract Review Topic Chat Introduction</u> is created by Reston
- A <u>Contract Review Topic Introductions To Chat</u> is composed of <u>Contract Review Topic Chat Introduction</u> elements
- A <u>Contract Review Topic Chat</u> is initiated by 1 <u>Contract Review Topic Chat Introduction</u>
- A <u>Contract Review Chat</u> is composed of 1 or more <u>Contract Review Topic Chat</u>
- A <u>Contract Review</u> is composed of 1 or more <u>Contract Review Chat</u>s
- A <u>Contract Creation Prompt</u> version is created using 1 <u>Contract Review</u>
- A <u>Contract Creation Prompt</u> version creates 1 <u>Contract</u> version+1 and 1 <u>Contract Discussion</u> version+1

## Process

1. In the course of reading the last version of the Contract and Contract Discussion created, Reston writes the topic title and text of a Contract Review Topic Chat Introduction in the set of Contract Review Topic Introductions To Chat that have not yet been submitted to chat with Claude.
2. When there is at least one Contract Review Topic to be discussed and Reston is ready to conduct a new Contract Review Chat with Claude, Reston performs the following:
    1. Position the window displaying Claude UI next to the window displaying the text/markdown editor of choice.
    2. In the text / markdown editor, open the file of topics available to chat about named "contract_review_topic_introductions_to_chat.md".  Select the text of the topic to be added to the chat and cut it from the file.
    3. In the window displaying the Claude text-based UI under the development project (the Supermarket Operations project in this case):
        1. Create a new chat (aka Conversation) as follows:
            1. Read dev/prompts/contract_review_chat_PROMPT.txt from a fresh clone of the SCHEME2 repo.
            2. Paste the text of the topic introduction in the initial message box presented in the conversation.
        2. Change the title of the conversation to a title of the form "Contract Review Chat_<contract_version>_<date_started>", where <date_started> is in the MMDDYYYY format.
        3. Send the message to Claude.
        4. Claude responds
        5. Continue the exchange with Claude on this topic or move on to the next topic or end the chat.
        6. Upon completing the last topic to be covered in the chat, tell Claude to:
            1. Format the text of this chat per the prompt given at the beginning of the chat session, to be downloaded by Reston.
    4. Reston puts the file at the directory at dev/contracts/contract_review_chats/, commits it and pushes it to the origin.

   