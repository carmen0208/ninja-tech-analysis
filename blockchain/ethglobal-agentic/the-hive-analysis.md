# The Hive: Web3 AI Agent Platform Analysis

## Introduction
This document contains an analysis of The [Hive platform](https://github.com/ask-the-hive/the-hive), exploring its architecture, AI agent system, and key features.

## Main Features of The Hive

### 1. AI-Powered Trading Agents
The application implements various AI agents specialized in different aspects of cryptocurrency trading and management:

- **Trading Agent**: Executes trades on Solana DEXs like Jupiter, Raydium, etc.
- **Liquidity Agent**: Manages liquidity provision and withdrawal from pools
- **Market Agent**: Provides market intelligence, trending tokens, and smart money flows
- **Token Analysis Agent**: Deep analysis of tokens including price charts, holders, and bubble maps
- **Wallet Agent**: Manages wallet operations, balances, and transfers
- **Staking Agent**: Handles staking and unstaking operations with yield data
- **Social Agent**: Integrates with social media (Twitter) for sentiment analysis

### 2. Web3 Authentication & Wallet Integration
- Uses Privy for web3 authentication and wallet connections
- Supports Solana wallet interactions
- Profile management and settings

### 3. Token Explorer and Analysis
- Detailed token data and analytics
- Price charts and historical data
- Holder analysis and distribution
- Top holder tracking

### 4. Portfolio Management
- Asset tracking and visualization
- Performance monitoring
- Balance tracking across tokens

### 5. Chat Interface
- AI-assisted conversational interface
- Natural language processing for crypto operations
- Guided trading, staking, and market analysis

### 6. DeFi Protocol Integrations
The platform integrates with numerous Solana DeFi protocols:
- **DEXs**: Jupiter, Raydium, Orca
- **Liquidity Pools**: Meteora, Raydium
- **Streaming Payments**: Streamflow
- **Staking**: Various liquid staking protocols

### 7. Data Providers & Analytics
Extensive integrations with data providers:
- **Helius**: For Solana blockchain data
- **Hellomoon**: For market data
- **Birdeye**: For trading data and analytics
- **Arkham**: For on-chain analytics
- **Twitter**: For social sentiment

### 8. Technical Architecture
- **Frontend**: React/Next.js with modern UI components (Radix UI)
- **State Management**: React Hooks and Contexts
- **Database**: Cosmos DB
- **Storage**: Azure Blob Storage
- **Search**: Azure Search
- **Analytics**: PostHog

### 9. Visualization Tools
- Price charts using Lightweight Charts
- Network graphs for token relationships
- Interactive data visualizations with Recharts
- Flow diagrams with XYFlow

## AI-Powered Trading Agents in The Hive

### Agent Structure and Location

Each agent is defined in the `ai/agents/[agent-name]` directory with a similar structure:
- `index.ts` - Main agent definition
- `capabilities.ts` - Description of agent capabilities
- `description.ts` - Detailed agent description
- `name.ts` - Agent name constant
- `tools.ts` - Tools/actions available to the agent

### How Users Call the Agents

Users interact with agents through a unified chat interface at `/app/(app)/chat`. The system uses:

1. **Natural Language Processing**: Users type natural language queries in the chat interface.
2. **Agent Routing**: Based on user intent, the system routes the query to the appropriate agent.
3. **Tool Selection**: The agent selects appropriate tools/actions to fulfill the request.
4. **Stream-Based UI**: Responses are streamed back to the user with updates as actions are performed.

The chat system uses context providers (`app/(app)/chat/_contexts/chat.tsx`) to manage state and API calls to `/api/chat` to process agent interactions on the server.

### Agent Capabilities

#### 1. Trading Agent (`ai/agents/trading`)
- **Location**: `ai/agents/trading/`
- **Abilities**: 
  - Execute trades on Solana DEXs (primarily Jupiter for best routing)
  - Look up token addresses
- **Key Tools**:
  - `SolanaTradeAction`: Executes trades between tokens
  - `SolanaGetTokenAddressAction`: Resolves token symbols to addresses
- **Definition**: Tools defined in `ai/solana/actions` with implementations in function.ts files

#### 2. Liquidity Agent (`ai/agents/liquidity`)
- **Location**: `ai/agents/liquidity/`
- **Abilities**:
  - Deposit liquidity to pools (Raydium, Orca, Meteora)
  - Withdraw liquidity from pools
  - Fetch pool information
  - Get LP token data
  - Get wallet addresses
- **Key Tools**:
  - `SolanaDepositLiquidityAction`: Add liquidity to pools
  - `SolanaWithdrawLiquidityAction`: Remove liquidity
  - `SolanaGetPoolsAction`: Get pool information
  - `SolanaGetLpTokensAction`: Get LP token holdings
- **Definition**: Implementations in `ai/solana/actions` directories

#### 3. Market Agent (`ai/agents/market`)
- **Location**: `ai/agents/market/`
- **Abilities**:
  - Get trending tokens on Solana
  - Identify top traders
  - Get trader transaction history
  - Track smart money inflows
- **Key Tools**:
  - `SolanaGetTrendingTokensAction`: Lists hot tokens
  - `SolanaGetTopTradersAction`: Identifies successful traders
  - `SolanaGetTraderTradesAction`: Gets trade history for specific accounts
  - `SolanaGetSmartMoneyInflowsAction`: Tracks institutional money flows
- **Definition**: Market data pulled from services like Birdeye, Hellomoon, and Helius

#### 4. Token Analysis Agent (`ai/agents/token-analysis`)
- **Location**: `ai/agents/token-analysis/`
- **Abilities**:
  - Get token metadata and price information
  - Analyze token holder distribution
  - Generate token price charts
  - Create bubble maps showing relationships
  - Track top traders for specific tokens
- **Key Tools**:
  - `SolanaGetTokenDataAction`: Get token details
  - `SolanaTopHoldersAction`: Get largest wallet holders
  - `SolanaTokenPriceChartAction`: Generate price charts
  - `SolanaGetBubbleMapsAction`: Visualize token relationships
- **Definition**: Analysis logic in `ai/solana/actions/token` directories

#### 5. Wallet Agent (`ai/agents/wallet`)
- **Location**: `ai/agents/wallet/`
- **Abilities**:
  - Get wallet addresses
  - Check SOL and token balances
  - Transfer tokens and SOL
- **Key Tools**:
  - `SolanaGetWalletAddressAction`: Get user's wallet address
  - `SolanaBalanceAction`: Check SOL balance
  - `SolanaAllBalancesAction`: Get all token balances
  - `SolanaTransferAction`: Send SOL or tokens
- **Definition**: Wallet operations defined in `ai/solana/actions` folders

#### 6. Staking Agent (`ai/agents/staking`)
- **Location**: `ai/agents/staking/`
- **Abilities**:
  - Stake SOL to validators
  - Unstake SOL from validators
  - Compare liquid staking yields
  - Get token addresses for staking derivatives
- **Key Tools**:
  - `SolanaStakeAction`: Stake SOL
  - `SolanaUnstakeAction`: Unstake SOL
  - `SolanaLiquidStakingYieldsAction`: Compare yields
- **Definition**: Staking functions in `ai/solana/actions/staking` directories

#### 7. Social Agent (`ai/agents/social`)
- **Location**: `ai/agents/social/`
- **Abilities**:
  - Search recent Twitter posts
  - Analyze social sentiment
- **Key Tools**:
  - `TwitterSearchRecentAction`: Fetch recent tweets
- **Definition**: Twitter API integrations via `ai/twitter` directory

### Integration with External Services

These agents leverage various external services:
- **DEXs**: Jupiter, Raydium, Orca for trading and liquidity
- **Data Providers**: Helius, Hellomoon, Birdeye for market data
- **Social APIs**: Twitter for sentiment analysis
- **Blockchain**: Solana RPC endpoints for on-chain interaction

### Technical Implementation

The agent architecture uses:
1. **OpenAI's function calling API**: To route intents to specific tools
2. **Vercel AI SDK**: For streaming responses and handling multi-turn conversations
3. **Custom Action Classes**: Each tool is implemented as a class with schema validation
4. **Connection Pooling**: Shared Solana RPC connections across tools

This architecture allows the AI to understand user intent, select the appropriate agent, choose the right tools, execute blockchain operations, and present the results to the user in a conversational interface.

## Agent Routing System in The Hive

### Frontend (Chat Input to API)

When a user sends a message in the chat interface, the flow starts in `app/(app)/chat/_contexts/chat.tsx`:

```typescript
const onSubmit = async () => {
    if (!input.trim()) return;
    setIsResponseLoading(true);
    await append({
        role: 'user',
        content: input,
    });
    setInput('');
}
```

The message is sent to the API endpoint via the `useAiChat` hook:

```typescript
const { messages, input, setInput, append, isLoading } = useAiChat({
    maxSteps: 20,
    onResponse: () => {
        setIsResponseLoading(false);
    },
    api: '/api/chat/solana', // This is the API endpoint that processes messages
    body: {
        model,
        modelName: model,
        userId: user?.id,
        chatId,
    },
});
```

### Backend Agent Selection (The Orchestrator)

The core of the agent routing happens in the `/api/chat/solana/route.ts` and `/api/chat/solana/utils.ts` files:

1. **Agent Selection Process:**

```typescript
// In route.ts
const chosenAgent = await chooseAgent(model, truncatedMessages);
```

2. **The Agent Selection Logic:**

```typescript
// In utils.ts
export const chooseAgent = async (model: LanguageModelV1, messages: Message[]): Promise<Agent | null> => {
    const { object } = await generateObject({
        model, // The AI model (OpenAI, Anthropic, etc.)
        schema: z.object({
            agent: z.enum(agents.map(agent => agent.name) as [string, ...string[]])
        }),
        messages, // The user's messages
        system  // System prompt that describes each agent's capabilities
    })

    return agents.find(agent => agent.name === object.name) ?? null;
}
```

3. **The System Prompt for Agent Selection:**

```typescript
// In utils.ts
export const system = 
`You are the orchestrator of a swarm of blockchain agents that each have specialized tasks.

Given this list of agents and their capabilities, choose the one that is most appropriate for the user's request.

${agents.map(agent => `${agent.name}: ${agent.systemPrompt}`).join("\n")}`
```

### Response Handling Based on Agent Selection

Once an agent is selected (or not), the system handles it in `route.ts`:

```typescript
// If no specific agent was chosen, use a generic response
if(!chosenAgent) {
    streamTextResult = streamText({
        model,
        messages: truncatedMessages,
        system, // Generic system prompt
    });
} else {
    // If an agent was chosen, use its specific tools and system prompt
    streamTextResult = streamText({
        model,
        tools: chosenAgent.tools, // The agent's specialized tools
        messages: truncatedMessages,
        system: `${chosenAgent.systemPrompt}\n\nUnless explicitly stated...`,
    });
}
```

### Key Components of the Agent Routing System

1. **The Orchestrator Pattern**: The system uses an "orchestrator" approach, where a top-level AI first classifies the request to determine which specialized agent should handle it.

2. **AI-Powered Routing**: The routing is not rule-based but uses AI understanding to match user intent to the most appropriate agent.

3. **Agent Registry**: All agents are registered in the `ai/agents/index.ts` file and imported into the routing system.

4. **Agent Interface**: Each agent follows a standard interface defined in `ai/agent.ts`:
   ```typescript
   export interface Agent {
       name: string;        // Unique agent name
       slug: string;        // URL-friendly identifier
       systemPrompt: string; // Detailed instructions
       capabilities: string; // Brief description of abilities
       tools: Record<string, CoreTool>; // Available actions
   }
   ```

5. **Dynamic Tool Selection**: After selection, only the chosen agent's tools are made available to the AI model, focusing it on the relevant capabilities.

This architecture allows the system to:
1. Understand user intent through natural language
2. Route to specialized agents based on the request type
3. Use only the tools relevant to that specific domain
4. Provide a cohesive user experience despite the underlying complexity

The system effectively creates a "swarm" of specialized AI agents that collaborate to handle different types of blockchain queries and actions, all orchestrated through a centralized routing mechanism.

## How The Hive Executes Agent Actions

### 1. Agent Invocation Flow

After the system selects the appropriate agent based on user intent, here's how it invokes the agent and processes the request:

#### Server-Side Processing (API Route)

1. **Agent Tool Configuration**:
   ```typescript
   // In route.ts
   streamTextResult = streamText({
       model,
       tools: chosenAgent.tools,  // The selected agent's specialized tools
       messages: truncatedMessages,
       system: `${chosenAgent.systemPrompt}\n\nUnless explicitly stated...`,
   });
   ```
   
   This code gives the AI model access to the specific tools of the chosen agent (e.g., trading tools for the Trading Agent). The selected agent's system prompt is also provided to guide the AI's behavior.

2. **Tool Definition Structure**:
   Each tool follows a pattern defined in `ai/solana/ai-sdk.ts`:
   ```typescript
   export const solanaTool = <TActionSchema, TResultBody>(
       action: SolanaAction<TActionSchema, TResultBody>, 
       connection: Connection
   ) => {
       return tool({
           description: action.description,
           parameters: action.argsSchema,
           execute: async (args) => {
               const result = func.length === 2 
                   ? await func(connection, args)
                   : await func(args);
               return result;
           }
       });
   }
   ```

3. **Stream-Based Response**:
   ```typescript
   return streamTextResult.toDataStreamResponse();
   ```
   The system uses streaming to send the response back to the client in real-time, allowing for progressive updates as the AI processes the request.

### 2. Client-Side Handling

#### Processing Tool Calls

1. **Tool Call Recognition**:
   In `app/(app)/chat/_components/messages.tsx`, the system renders messages with tool invocations:
   ```typescript
   <Message 
       key={message.id} 
       message={message} 
       className={messageClassName} 
       ToolComponent={ToolInvocation}  // This renders any tool calls in the message
       previousMessage={index > 0 ? messages[index - 1] : undefined} 
       nextMessage={index < messages.length - 1 ? messages[index + 1] : undefined} 
   />
   ```

2. **Tool Dispatch**:
   In `app/(app)/chat/_components/tools/index.tsx`, the system routes to specific tool implementations:
   ```typescript
   const ToolInvocation: React.FC<Props> = ({ tool, prevToolAgent }) => {
       const toolParts = tool.toolName.split("-");
       const toolName = toolParts.slice(1).join("-");
       
       switch(toolName) {
           case SOLANA_TRADE_NAME:
               return <Trade tool={tool} prevToolAgent={prevToolAgent} />
           // Other tool cases...
       }
   }
   ```

3. **Tool UI Components**:
   Each tool has a dedicated UI component that:
   - Shows a call UI with parameters
   - Executes the operation
   - Reports results back through the chat context

### 3. Example Flow: Trading Process

Let's follow the complete flow for a trading operation:

1. **User Input**:
   User types: "I want to swap 10 SOL for USDC"

2. **Agent Selection**:
   The orchestrator recognizes this as a trading request and selects the Trading Agent

3. **Tool Selection**:
   The Trading Agent identifies the need to use the `SolanaTradeAction` tool

4. **Tool Invocation**:
   ```typescript
   // The AI model invokes the trading tool with parameters
   {
       inputMint: "So11111111111111111111111111111111111111112", // SOL address
       outputMint: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", // USDC address
       inputAmount: 10
   }
   ```

5. **UI Rendering**:
   The swap component is rendered with these parameters:
   ```typescript
   <SwapCallBody toolCallId={toolCallId} args={args} />
   ```

6. **Transaction Execution**:
   ```typescript
   const onSwap = async () => {
       if(!wallet || !quoteResponse) return;
       setIsSwapping(true);
       try {
           // Get swap transaction from Jupiter
           const { swapTransaction} = await getSwapObj(wallet.address, quoteResponse);
           const swapTransactionBuf = Buffer.from(swapTransaction, "base64");
           const transaction = VersionedTransaction.deserialize(swapTransactionBuf);
           // Send transaction via user's wallet
           const txHash = await sendTransaction(transaction);
           onSuccess?.(txHash);
       } catch (error) {
           onError?.(error instanceof Error ? error.message : "Unknown error");
       } finally {
           setIsSwapping(false);
       }
   }
   ```

7. **Result Reporting**:
   The result is reported back to the chat context:
   ```typescript
   addToolResult<SolanaTradeResultBodyType>(toolCallId, {
       message: `Swap successful!`,
       body: {
           transaction: tx,
           inputAmount: args.inputAmount || 0,
           inputToken: inputTokenData?.symbol || "",
           outputToken: outputTokenData?.symbol || "",
       }
   });
   ```

8. **AI Continuation**:
   The AI continues the conversation with knowledge of the action result:
   ```
   "I've successfully swapped 10 SOL for approximately 240 USDC. The transaction was confirmed on the Solana blockchain with hash 0x123..."
   ```

### 4. Key Architecture Patterns

1. **Tool-Based Agent Architecture**: 
   Agents are defined as collections of specialized tools rather than complete isolated systems.

2. **Client-Side Tool Execution**:
   Many blockchain operations happen client-side in the browser to allow for wallet signing.

3. **Real-Time UI Updates**:
   The system uses a streaming API and tool result reporting to provide real-time UI updates.

4. **Unified Chat Interface**:
   Despite the complexity of many different agents and tools, everything is presented through a single chat interface.

5. **Context Preservation**:
   The `prevToolAgent` parameter helps maintain context across different tool invocations.

This architecture allows the AI to understand user intent, route to the correct specialized agent, invoke the appropriate blockchain actions, and maintain an ongoing conversation—all while keeping the user informed through an intuitive chat interface.

## Multi-Step Tool and Agent Interactions in The Hive

### Key Components for Multi-Step Processing

#### 1. `maxSteps` Parameter in `useAiChat`

The critical component that enables multi-step processing is found in the chat context:

```typescript
const { messages, input, setInput, append, isLoading, addToolResult: addToolResultBase, setMessages } = useAiChat({
    maxSteps: 20,  // Allows up to 20 steps of tool calls before returning to the user
    onResponse: () => {
        setIsResponseLoading(false);
    },
    api: '/api/chat/solana',
    body: {
        model,
        modelName: model,
        userId: user?.id,
        chatId,
    },
});
```

**How it works**: 
- The `maxSteps: 20` configuration allows the AI to make up to 20 consecutive tool calls within a single user interaction.
- Without needing to return control to the user between steps, the AI can execute multiple tools in sequence.
- This is crucial for complex operations that require multiple tools to be called in a specific order.

#### 2. Tool Result Handling

The system has a mechanism to feed tool results back to the AI for continued processing:

```typescript
const addToolResult = <T,>(toolCallId: string, result: ToolResult<T>) => {
    addToolResultBase({
        toolCallId,
        result,
    })
}
```

**How it works**:
- When a tool completes its operation, the result is fed back to the AI model.
- The AI can then decide to make another tool call based on this result.
- This creates a back-and-forth between the AI and tools without user intervention.

#### 3. Streaming Architecture

The application uses streaming responses that allow for real-time updates:

```typescript
streamTextResult = streamText({
    model,
    tools: chosenAgent.tools,
    messages: truncatedMessages,
    system: `${chosenAgent.systemPrompt}\n\nUnless explicitly stated...`,
});

return streamTextResult.toDataStreamResponse();
```

**How it works**:
- The streaming architecture allows the UI to update in real-time as each tool completes.
- Users see the progress of complex multi-step operations rather than waiting for the entire sequence to complete.

### 4. Agent Invocation Tool

The system includes a special `InvokeAgent` tool that allows agents to call other agents:

```typescript
// InvokeAgent component
return (
    <ToolCard 
        tool={tool}
        loadingText={`Invoking Agent...`}
        result={{
            heading: (result: InvokeAgentResultType) => result.body 
                ? `Invoked ${tool.args.agent}` 
                : `Failed to invoke agent`,
            body: (result: InvokeAgentResultType) => result.body 
                ? result.body.message
                : "Failed to invoke agent"
        }}
        defaultOpen={true}
        prevToolAgent={prevToolAgent}
    />
)
```

**How it works**:
- One agent can delegate tasks to another more specialized agent.
- The `prevToolAgent` parameter tracks the chain of agent invocations to maintain context.
- This enables a hierarchical agent architecture where agents can collaborate on complex tasks.

### Example Multi-Step Workflow

Here's how a complex user request might be processed:

1. **User Request**: "I want to swap 10 SOL for USDC and stake the resulting USDC on the platform with the highest yield"

2. **Agent Selection**: The orchestrator selects the Trading Agent based on the initial request

3. **Step 1**: Trading Agent uses `SolanaGetTokenAddressAction` to get addresses for SOL and USDC

4. **Step 2**: Trading Agent uses `SolanaTradeAction` to perform the swap

5. **Step 3**: Instead of returning to the user, the Trading Agent uses `InvokeAgentAction` to call the Staking Agent

6. **Step 4**: Staking Agent uses `SolanaLiquidStakingYieldsAction` to find the best staking platform

7. **Step 5**: Staking Agent uses `SolanaStakeAction` to stake the USDC

8. **Final Response**: The AI consolidates all these steps into a single coherent response to the user

Throughout this process:
- All steps appear in the UI as they complete
- The user sees real-time progress
- The AI maintains context across all operations
- A comprehensive final response ties everything together

### Technical Implementation Details

1. **Vercel AI SDK Integration**:
   - The Hive leverages Vercel's AI SDK's multi-step capabilities
   - The `maxSteps` parameter controls how many tool calls can be made in sequence

2. **Global vs. Agent-Specific Context**:
   - The `prevToolAgent` parameter helps maintain context between different agents
   - This ensures continuity when one agent invokes another

3. **Task Chaining Pattern**:
   - The AI SDK automatically chains tool results back into the conversation
   - This maintains a coherent understanding of the entire operation

4. **Fallback Mechanism**:
   - If any step fails, the system can provide a graceful explanation
   - The AI can recommend alternative approaches based on which step failed

This architecture enables The Hive to handle complex multi-step workflows that require coordination between different specialized agents, all while providing a seamless experience through a single chat interface.
