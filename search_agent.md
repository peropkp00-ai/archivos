import OpenAI from 'openai';
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";
import fs from 'fs';
import path from 'path';
import { logger } from './logger.js';
import "dotenv/config";

/**
 * @fileoverview Lógica del agente buscador individual.
 * Implementa "Structured Outputs" para garantizar la ejecución fiable de herramientas
 * y el cumplimiento de reglas estrictas (ej. límite de palabras).
 */

// --- Esquema JSON para Salida Estructurada ---
const SEARCH_ACTION_SCHEMA = {
  name: "search_action",
  strict: true,
  schema: {
    type: "object",
    properties: {
      thought: {
        type: "string",
        description: "Razonamiento breve sobre qué buscar y por qué."
      },
      tool_name: {
        type: "string",
        enum: ["buscar_ley", "obtener_contenido_completo_ley", "finalizar"],
        description: "La herramienta a ejecutar."
      },
      arguments: {
        type: "object",
        properties: {
          termino: {
            type: "string",
            description: "Palabras clave para buscar_ley (MÁXIMO 2 PALABRAS)."
          },
          id_norma: {
            type: "string",
            description: "ID de la norma para obtener_contenido_completo_ley."
          },
          id_ley: {
            type: "string",
            description: "ID de la ley (alternativo)."
          }
        },
        additionalProperties: false
      }
    },
    required: ["thought", "tool_name", "arguments"],
    additionalProperties: false
  }
};

// --- Lógica del Agente Buscador ---

function logAgentFailure(modelName, reason, lastToolCall = "N/A", lastToolResponse = "N/A") {
  const logDir = path.join(process.cwd(), 'logs');
  if (!fs.existsSync(logDir)) {
    fs.mkdirSync(logDir);
  }

  const sessionTimestamp = logger.sessionTimestamp || new Date().toISOString().replace(/[:.]/g, '-');
  const logFile = path.join(logDir, `agent_failures-${sessionTimestamp}.log`);
  const timestamp = new Date().toISOString();

  const logEntry = `
=== FALLO AGENTE [${timestamp}] ===
Modelo: ${modelName}
Razón: ${reason}
Última Herramienta: ${lastToolCall}
Respuesta Herramienta: ${lastToolResponse}
================================
`;

  fs.appendFileSync(logFile, logEntry);
}

export async function runSearchAgent(instruction, context, baseSystemPrompt, models, apiKeys) {
  const primaryModel = models[0];
  if (!apiKeys || (Array.isArray(apiKeys) && apiKeys.length === 0)) {
    logger.error(`[Buscador - ${primaryModel}] Error: No se proporcionaron claves de API.`);
    return { model: primaryModel, evidence: "", reason: "No se proporcionaron claves de API." };
  }

  logger.info(`[Buscador - ${primaryModel} (+fallback)] Iniciando investigación con Structured Outputs...`);
  logger.debug(`[Buscador - ${primaryModel}] 🎯 Instrucción Activa: "${instruction.substring(0, 100)}..."`);

  const mcpClient = new Client({ name: `search-agent-client-${primaryModel.replace(/[^a-zA-Z0-9]/g, '_')}`, version: "1.0.0" });
  let collectedText = "";

  try {
    const transport = new SSEClientTransport(
      new URL("http://127.0.0.1:8090/sse")
    );

    await mcpClient.connect(transport);
    logger.info(`[Buscador - ${primaryModel}] 🔌 Conectado al servidor MCP.`);

    // Listamos herramientas solo para verificar conexión, pero usamos el esquema estático
    await mcpClient.listTools();

    // Construcción del System Prompt
    const systemPrompt = `${baseSystemPrompt}
    
    === CONTEXTO DE REFERENCIA ===
    Consulta Original: "${context.originalQuery}"
    
    === INSTRUCCIONES DE FORMATO ===
    Debes responder SIEMPRE con un objeto JSON válido que siga este esquema:
    {
      "thought": "Tu razonamiento breve...",
      "tool_name": "buscar_ley" | "obtener_contenido_completo_ley" | "finalizar",
      "arguments": { ... }
    }
    
    REGLA CRÍTICA: Para 'buscar_ley', el argumento 'termino' debe tener MÁXIMO 2 PALABRAS.
    `;

    let messages = [
      { role: "system", content: systemPrompt },
      { role: "user", content: instruction }
    ];

    // Bucle de interacción (máximo 5 turnos)
    for (let i = 0; i < 5; i++) {
      logger.info(`[Buscador - ${primaryModel}] 🔄 Iteración ${i + 1}/5`);

      let responseContent = null;
      let lastError = null;
      const keysToTry = Array.isArray(apiKeys) ? apiKeys : [apiKeys];

      // Rotación de API Keys
      for (let k = 0; k < keysToTry.length; k++) {
        const currentKey = keysToTry[k];
        try {
          const currentOpenAI = new OpenAI({
            baseURL: 'https://openrouter.ai/api/v1',
            apiKey: currentKey,
          });

          if (k > 0) logger.warn(`[Buscador - ${primaryModel}] ⚠️ Reintentando con Key alternativa...`);

          const completion = await currentOpenAI.chat.completions.create({
            model: models[0],
            messages: messages,
            temperature: 0.2, // Baja temperatura para precisión
            response_format: {
              type: "json_schema",
              json_schema: SEARCH_ACTION_SCHEMA
            }
          });

          responseContent = completion.choices[0].message.content;
          break; // Éxito

        } catch (error) {
          lastError = error;
          logger.warn(`[Buscador - ${primaryModel}] 🛑 Error con Key ${k}: ${error.message}`);
          continue;
        }
      }

      if (!responseContent) {
        throw lastError || new Error("Fallaron todas las API Keys.");
      }

      // Procesar Respuesta Estructurada
      let action;
      try {
        action = JSON.parse(responseContent);
        logger.debug(`[Buscador - ${primaryModel}] 🧠 Pensamiento: ${action.thought}`);
      } catch (e) {
        logger.error(`[Buscador - ${primaryModel}] ❌ Error parseando JSON del modelo: ${responseContent}`);
        // Inyectar error al modelo para que se corrija
        messages.push({ role: "user", content: "Error: Tu respuesta no fue un JSON válido. Intenta de nuevo siguiendo el esquema." });
        continue;
      }

      // Ejecutar Acción
      if (action.tool_name === 'finalizar') {
        logger.info(`[Buscador - ${primaryModel}] 🏁 El modelo decidió finalizar.`);
        break;
      }

      if (action.tool_name === 'buscar_ley') {
        let termino = action.arguments.termino;

        // --- REGLA DE SEGURIDAD: TRUNCAR A 2 PALABRAS ---
        if (termino) {
          const words = termino.trim().split(/\s+/);
          if (words.length > 2) {
            const truncated = words.slice(0, 2).join(" ");
            logger.warn(`[Buscador - ${primaryModel}] ⚠️ Término "${termino}" excede 2 palabras. Truncando a: "${truncated}"`);
            termino = truncated;
          }
        }

        logger.info(`[Buscador - ${primaryModel}] 🛠️ Ejecutando buscar_ley: "${termino}"`);
        const toolResult = await mcpClient.callTool({
          name: "buscar_ley",
          arguments: { termino: termino }
        });

        // Inyectar resultado como mensaje de usuario (simulación de tool output)
        const resultText = JSON.stringify(toolResult);
        messages.push({ role: "assistant", content: responseContent }); // Guardar lo que dijo el modelo
        messages.push({ role: "user", content: `Resultado de buscar_ley: ${resultText}` });
      }

      else if (action.tool_name === 'obtener_contenido_completo_ley') {
        const idNorma = action.arguments.id_norma || action.arguments.id_ley;
        logger.info(`[Buscador - ${primaryModel}] 🛠️ Ejecutando obtener_contenido_completo_ley: ID ${idNorma}`);

        const toolResult = await mcpClient.callTool({
          name: "obtener_contenido_completo_ley",
          arguments: { id_norma: idNorma, id_ley: idNorma }
        });

        const contentText = toolResult?.content?.[0]?.text;
        let parsedContent = null;
        try {
          parsedContent = JSON.parse(contentText);
        } catch (e) { }

        if (parsedContent?.contenido) {
          let finalContent = parsedContent.contenido;
          if (finalContent.length > 1000000) {
            finalContent = finalContent.substring(0, 1000000) + "\n[TRUNCADO 1M chars]";
          }
          collectedText = `--- Fuente: ${primaryModel} encontró Ley ID ${idNorma} ---\n${finalContent}\n\n`;
          logger.info(`[Buscador - ${primaryModel}] ✅ EVIDENCIA ENCONTRADA.`);
          break; // Éxito total
        } else {
          messages.push({ role: "assistant", content: responseContent });
          messages.push({ role: "user", content: `Resultado: ${contentText}` });
        }
      }
    }

    // Captura Pasiva si no hay evidencia formal
    if (!collectedText) {
      const thoughts = messages
        .filter(m => m.role === 'assistant')
        .map(m => {
          try { return JSON.parse(m.content).thought; } catch (e) { return ""; }
        })
        .filter(t => t)
        .join("\n");

      if (thoughts) {
        collectedText = `--- EVIDENCIA PARCIAL (Pensamientos) ---\n${thoughts}\n`;
      }
    }

  } catch (e) {
    logger.error(`[Buscador - ${primaryModel}] Error fatal: ${e.message}`);
    return { model: primaryModel, evidence: collectedText, reason: e.message, failed: true };
  } finally {
    await mcpClient.close();
  }

  return { model: primaryModel, evidence: collectedText, failed: false };
}
