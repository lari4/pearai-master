# Документация AI Промтов PearAI

Этот документ содержит полную документацию всех AI промтов, используемых в приложении PearAI, сгруппированных по тематикам.

## Содержание

1. [Системные Промты](#системные-промты)
2. [Промты Целей и Задач](#промты-целей-и-задач)
3. [Промты Возможностей](#промты-возможностей)
4. [Промты Правил](#промты-правил)
5. [Промты Работы с Инструментами](#промты-работы-с-инструментами)
6. [Промты Режимов Работы](#промты-режимов-работы)
7. [Промты MCP Серверов](#промты-mcp-серверов)
8. [Промты Инструментов](#промты-инструментов)
9. [Промты Пользовательских Инструкций](#промты-пользовательских-инструкций)

---

## Системные Промты

### 1.1 Главный Системный Промт (System Prompt Generator)

**Назначение:** Это основной промт-генератор, который собирает все компоненты системного промта в единое целое. Он отвечает за создание полного контекста для AI агента в зависимости от выбранного режима работы.

**Расположение:** `src/core/prompts/system.ts`

**Как работает:**
- Принимает параметры: контекст расширения, рабочую директорию, режим работы, конфигурации
- Проверяет наличие пользовательского промта в файле `.roo/system-prompt-<mode_slug>`
- Если пользовательский промт найден, использует его
- Иначе генерирует промт из стандартных секций
- Добавляет кастомные инструкции в конце

**Структура генерируемого промта:**
1. Определение роли (roleDefinition)
2. Секция использования инструментов (Tool Use)
3. Описание инструментов для режима
4. Руководство по использованию инструментов
5. Информация о MCP серверах (если доступны)
6. Возможности агента
7. Доступные режимы
8. Правила работы
9. Системная информация
10. Цели и задачи
11. Кастомные инструкции

**Код:**

```typescript
async function generatePrompt(
	context: vscode.ExtensionContext,
	cwd: string,
	supportsComputerUse: boolean,
	mode: Mode,
	mcpHub?: McpHub,
	diffStrategy?: DiffStrategy,
	browserViewportSize?: string,
	promptComponent?: PromptComponent,
	customModeConfigs?: ModeConfig[],
	globalCustomInstructions?: string,
	diffEnabled?: boolean,
	experiments?: Record<string, boolean>,
	enableMcpServerCreation?: boolean,
	language?: string,
	rooIgnoreInstructions?: string,
): Promise<string> {
	if (!context) {
		throw new Error("Extension context is required for generating system prompt")
	}

	// If diff is disabled, don't pass the diffStrategy
	const effectiveDiffStrategy = diffEnabled ? diffStrategy : undefined

	// Get the full mode config to ensure we have the role definition
	const modeConfig = getModeBySlug(mode, customModeConfigs) || modes.find((m) => m.slug === mode) || modes[0]
	const roleDefinition = promptComponent?.roleDefinition || modeConfig.roleDefinition

	const [modesSection, mcpServersSection] = await Promise.all([
		getModesSection(context),
		modeConfig.groups.some((groupEntry) => getGroupName(groupEntry) === "mcp")
			? getMcpServersSection(mcpHub, effectiveDiffStrategy, enableMcpServerCreation)
			: Promise.resolve(""),
	])

	const basePrompt = `${roleDefinition}

${getSharedToolUseSection()}

${getToolDescriptionsForMode(
	mode,
	cwd,
	supportsComputerUse,
	effectiveDiffStrategy,
	browserViewportSize,
	mcpHub,
	customModeConfigs,
	experiments,
)}

${getToolUseGuidelinesSection()}

${mcpServersSection}

${getCapabilitiesSection(cwd, supportsComputerUse, mcpHub, effectiveDiffStrategy)}

${modesSection}

${getRulesSection(cwd, supportsComputerUse, effectiveDiffStrategy)}

${getSystemInfoSection(cwd)}

${getObjectiveSection()}

${await addCustomInstructions(promptComponent?.customInstructions || modeConfig.customInstructions || "", globalCustomInstructions || "", cwd, mode, { language: language ?? formatLanguage(vscode.env.language), rooIgnoreInstructions })}`

	return basePrompt
}

export const SYSTEM_PROMPT = async (
	context: vscode.ExtensionContext,
	cwd: string,
	supportsComputerUse: boolean,
	mcpHub?: McpHub,
	diffStrategy?: DiffStrategy,
	browserViewportSize?: string,
	mode: Mode = defaultModeSlug,
	customModePrompts?: CustomModePrompts,
	customModes?: ModeConfig[],
	globalCustomInstructions?: string,
	diffEnabled?: boolean,
	experiments?: Record<string, boolean>,
	enableMcpServerCreation?: boolean,
	language?: string,
	rooIgnoreInstructions?: string,
): Promise<string> => {
	if (!context) {
		throw new Error("Extension context is required for generating system prompt")
	}

	const getPromptComponent = (value: unknown) => {
		if (typeof value === "object" && value !== null) {
			return value as PromptComponent
		}
		return undefined
	}

	// Try to load custom system prompt from file
	const variablesForPrompt: PromptVariables = {
		workspace: cwd,
		mode: mode,
		language: language ?? formatLanguage(vscode.env.language),
		shell: vscode.env.shell,
		operatingSystem: os.type(),
	}
	const fileCustomSystemPrompt = await loadSystemPromptFile(cwd, mode, variablesForPrompt)

	// Check if it's a custom mode
	const promptComponent = getPromptComponent(customModePrompts?.[mode])

	// Get full mode config from custom modes or fall back to built-in modes
	const currentMode = getModeBySlug(mode, customModes) || modes.find((m) => m.slug === mode) || modes[0]

	// If a file-based custom system prompt exists, use it
	if (fileCustomSystemPrompt) {
		const roleDefinition = promptComponent?.roleDefinition || currentMode.roleDefinition
		const customInstructions = await addCustomInstructions(
			promptComponent?.customInstructions || currentMode.customInstructions || "",
			globalCustomInstructions || "",
			cwd,
			mode,
			{ language: language ?? formatLanguage(vscode.env.language), rooIgnoreInstructions },
		)

		// For file-based prompts, don't include the tool sections
		return `${roleDefinition}

${fileCustomSystemPrompt}

${customInstructions}`
	}

	// If diff is disabled, don't pass the diffStrategy
	const effectiveDiffStrategy = diffEnabled ? diffStrategy : undefined

	return generatePrompt(
		context,
		cwd,
		supportsComputerUse,
		currentMode.slug,
		mcpHub,
		effectiveDiffStrategy,
		browserViewportSize,
		promptComponent,
		customModes,
		globalCustomInstructions,
		diffEnabled,
		experiments,
		enableMcpServerCreation,
		language,
		rooIgnoreInstructions,
	)
}
```

---

## Промты Целей и Задач

### 2.1 Objective Section - Секция Целей

**Назначение:** Определяет основную цель и методологию работы AI агента. Это ключевой промт, который устанавливает итеративный подход к решению задач.

**Расположение:** `src/core/prompts/sections/objective.ts`

**Ключевые инструкции:**
1. Итеративное выполнение задач с разбиением на шаги
2. Анализ задачи и установка достижимых целей
3. Последовательная работа с использованием инструментов
4. Обязательное использование attempt_completion для представления результата
5. Запрет на бесконечные диалоги

**Код:**

```typescript
export function getObjectiveSection(): string {
	return `====

OBJECTIVE

You accomplish a given task iteratively, breaking it down into clear steps and working through them methodically.

1. Analyze the user's task and set clear, achievable goals to accomplish it. Prioritize these goals in a logical order.
2. Work through these goals sequentially, utilizing available tools one at a time as necessary. Each goal should correspond to a distinct step in your problem-solving process. You will be informed on the work completed and what's remaining as you go.
3. Remember, you have extensive capabilities with access to a wide range of tools that can be used in powerful and clever ways as necessary to accomplish each goal. Before calling a tool, do some analysis within <thinking></thinking> tags. First, analyze the file structure provided in environment_details to gain context and insights for proceeding effectively. Then, think about which of the provided tools is the most relevant tool to accomplish the user's task. Next, go through each of the required parameters of the relevant tool and determine if the user has directly provided or given enough information to infer a value. When deciding if the parameter can be inferred, carefully consider all the context to see if it supports a specific value. If all of the required parameters are present or can be reasonably inferred, close the thinking tag and proceed with the tool use. BUT, if one of the values for a required parameter is missing, DO NOT invoke the tool (not even with fillers for the missing params) and instead, ask the user to provide the missing parameters using the ask_followup_question tool. DO NOT ask for more information on optional parameters if it is not provided.
4. Once you've completed the user's task, you must use the attempt_completion tool to present the result of the task to the user. You may also provide a CLI command to showcase the result of your task; this can be particularly useful for web development tasks, where you can run e.g. \`open index.html\` to show the website you've built.
5. The user may provide feedback, which you can use to make improvements and try again. But DO NOT continue in pointless back and forth conversations, i.e. don't end your responses with questions or offers for further assistance.`
}
```

