# Building AI-Powered Confluence Data Center Plugins: Complete Novel Editor Implementation Guide

This comprehensive guide covers creating a modern Confluence Data Center 8.5 plugin with an AI-powered inline editor similar to the Novel template, featuring proper frontend/backend separation, webpack bundling, and complete project structure.

## Plugin architecture foundations

Confluence Data Center 8.5 plugins follow a sophisticated **hybrid build pattern** where frontend components build first using webpack, then integrate seamlessly into the backend Maven build process.  This architecture enables modern JavaScript development while maintaining enterprise-grade compatibility.  

The **core build workflow** uses the frontend-maven-plugin to execute webpack builds during the `generate-resources` phase, before the main Atlassian Maven Plugin (AMPS) processes the backend components.  This ensures bundled JavaScript assets are available when the plugin loads.  

**Essential Maven configuration** includes specific Data Center compatibility markers in the atlassian-plugin.xml, with parameters like `atlassian-data-center-status="compatible"` and `atlassian-data-center-compatible="true"`.   The plugin structure supports clustering requirements through stateless design patterns, serializable caching, and cluster-safe development practices. 

## Complete project structure and configuration

### Maven POM.xml configuration

The foundation begins with a properly configured POM.xml that orchestrates the frontend-backend build process: 

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example.confluence</groupId>
    <artifactId>ai-editor-plugin</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>atlassian-plugin</packaging>
    
    <properties>
        <confluence.version>8.5.4</confluence.version>
        <amps.version>8.2.10</amps.version>
        <frontend-maven-plugin.version>1.12.1</frontend-maven-plugin.version>
        <node.version>v18.17.0</node.version>
        <npm.version>9.6.7</npm.version>
        <platform.version>7.0.9</platform.version>
    </properties>
    
    <build>
        <plugins>
            <!-- Frontend build first -->
            <plugin>
                <groupId>com.github.eirslett</groupId>
                <artifactId>frontend-maven-plugin</artifactId>
                <version>${frontend-maven-plugin.version}</version>
                <configuration>
                    <nodeVersion>${node.version}</nodeVersion>
                    <npmVersion>${npm.version}</npmVersion>
                    <workingDirectory>${project.basedir}/frontend</workingDirectory>
                </configuration>
                <executions>
                    <execution>
                        <id>install node and npm</id>
                        <goals><goal>install-node-and-npm</goal></goals>
                        <phase>generate-resources</phase>
                    </execution>
                    <execution>
                        <id>npm install</id>
                        <goals><goal>npm</goal></goals>
                        <phase>generate-resources</phase>
                        <configuration>
                            <arguments>install</arguments>
                        </configuration>
                    </execution>
                    <execution>
                        <id>webpack build</id>
                        <goals><goal>npm</goal></goals>
                        <phase>generate-resources</phase>
                        <configuration>
                            <arguments>run build</arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            
            <!-- Confluence Maven Plugin -->
            <plugin>
                <groupId>com.atlassian.maven.plugins</groupId>
                <artifactId>confluence-maven-plugin</artifactId>
                <version>${amps.version}</version>
                <extensions>true</extensions>
                <configuration>
                    <productVersion>${confluence.version}</productVersion>
                    <productDataVersion>${confluence.version}</productDataVersion>
                    <instructions>
                        <Atlassian-Plugin-Key>${project.groupId}.${project.artifactId}</Atlassian-Plugin-Key>
                        <Export-Package/>
                        <Import-Package>*;resolution:=optional</Import-Package>
                        <Atlassian-Scan-Folders>META-INF/plugin-descriptors</Atlassian-Scan-Folders>
                    </instructions>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Project directory structure

The **recommended project organization** separates concerns while maintaining clear build dependencies: 

```
ai-editor-confluence-plugin/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/plugin/
│   │   │       ├── macro/AIEditorMacro.java
│   │   │       ├── rest/AIController.java
│   │   │       └── service/AIService.java
│   │   └── resources/
│   │       ├── atlassian-plugin.xml
│   │       ├── templates/ai-editor-macro.vm
│   │       └── i18n/
│   └── test/java/
└── frontend/
    ├── package.json
    ├── webpack.config.js
    ├── tsconfig.json
    └── src/
        ├── index.tsx
        ├── components/
        │   ├── AIEditor.tsx  
        │   ├── NovelEditor.tsx
        │   └── AIAssistant.tsx
        └── styles/
```

### Atlassian plugin descriptor

The **atlassian-plugin.xml** serves as the plugin’s central configuration, defining all components and their relationships: 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<atlassian-plugin key="${project.groupId}.${project.artifactId}" 
                  name="AI Editor Plugin" 
                  plugins-version="2">
    
    <plugin-info>
        <description>AI-powered inline editor for Confluence</description>
        <version>${project.version}</version>
        <vendor name="Example Corp" url="https://example.com"/>
        
        <!-- Data Center compatibility markers -->
        <param name="atlassian-data-center-status">compatible</param>
        <param name="atlassian-data-center-compatible">true</param>
        
        <param name="plugin-icon">images/plugin-icon.png</param>
    </plugin-info>

    <!-- Internationalization -->
    <resource type="i18n" name="i18n" location="i18n.ai-editor"/>

    <!-- Component imports -->
    <component-import key="applicationProperties" 
                     interface="com.atlassian.sal.api.ApplicationProperties"/>
    <component-import key="templateRenderer" 
                     interface="com.atlassian.templaterenderer.TemplateRenderer"/>
    <component-import key="pluginSettingsFactory" 
                     interface="com.atlassian.sal.api.pluginsettings.PluginSettingsFactory"/>

    <!-- Spring configuration -->
    <component key="aiService" 
              class="com.example.plugin.service.AIService">
        <description>AI service for content generation</description>
    </component>

    <!-- Web resources with webpack output -->
    <web-resource key="ai-editor-resources" name="AI Editor Resources">
        <dependency>com.atlassian.auiplugin:ajs</dependency>
        <dependency>com.atlassian.auiplugin:aui-dialog2</dependency>
        
        <resource type="download" name="ai-editor.css" location="assets/ai-editor.css"/>
        <resource type="download" name="ai-editor.js" location="assets/ai-editor.js"/>
        
        <transformation extension="js">
            <transformer key="jsI18n"/>
        </transformation>
        
        <context>atl.general</context>
        <context>confluence.view.content</context>
        <context>confluence.edit.content</context>
    </web-resource>

    <!-- AI Editor Macro -->
    <macro key="ai-editor-macro" name="AI Editor" class="com.example.plugin.macro.AIEditorMacro">
        <description>AI-powered content editor</description>
        <resource type="velocity" name="macro" location="/templates/ai-editor-macro.vm"/>
        <parameters>
            <parameter name="model" type="enum" values="gpt-3.5-turbo,gpt-4">
                <description>AI model to use</description>
                <default>gpt-3.5-turbo</default>
            </parameter>
            <parameter name="prompt" type="string">
                <description>Initial prompt</description>
            </parameter>
        </parameters>
    </macro>

    <!-- REST API endpoints -->
    <rest key="ai-rest-api" path="/ai" version="1.0">
        <description>AI service REST endpoints</description>
    </rest>
    
</atlassian-plugin>
```

## Frontend implementation with Novel-style editor

### Webpack configuration for Confluence integration

The **webpack configuration** leverages the atlassian-webresource-webpack-plugin to seamlessly integrate modern JavaScript bundling with Confluence’s resource management: 

```javascript
const path = require('path');
const WrmPlugin = require('atlassian-webresource-webpack-plugin');

const FRONTEND_SRC_DIR = path.join(__dirname, 'src');
const OUTPUT_DIR = path.join(__dirname, '..', 'target', 'classes');

module.exports = {
    mode: 'development',
    entry: {
        'ai-editor': path.join(FRONTEND_SRC_DIR, 'index.tsx'),
    },
    
    output: {
        path: OUTPUT_DIR,
        filename: 'assets/[name].js',
        publicPath: '/'
    },
    
    plugins: [
        new WrmPlugin({
            pluginKey: 'com.example.confluence.ai-editor-plugin',
            contextMap: {
                'ai-editor': ['atl.general', 'confluence.view.content', 'confluence.edit.content']
            },
            xmlDescriptors: path.resolve(OUTPUT_DIR, 'META-INF', 'plugin-descriptors', 'wr-webpack-bundles.xml'),
            providedDependencies: {
                'react': {
                    dependency: 'com.atlassian.plugins.atlassian-react:react',
                    import: {
                        var: "require('react')",
                        amd: 'react'
                    }
                },
                'react-dom': {
                    dependency: 'com.atlassian.plugins.atlassian-react:react-dom',
                    import: {
                        var: "require('react-dom')",
                        amd: 'react-dom'
                    }
                }
            }
        })
    ],
    
    module: {
        rules: [
            {
                test: /\.(js|jsx|ts|tsx)$/,
                exclude: /node_modules/,
                use: {
                    loader: 'babel-loader',
                    options: {
                        presets: [
                            '@babel/preset-env',
                            '@babel/preset-react',
                            '@babel/preset-typescript'
                        ]
                    }
                }
            },
            {
                test: /\.css$/,
                use: ['style-loader', 'css-loader']
            }
        ]
    },
    
    resolve: {
        extensions: ['.js', '.jsx', '.ts', '.tsx']
    },
    
    externals: {
        'react': 'React',
        'react-dom': 'ReactDOM'
    }
};
```

### TypeScript configuration

**TypeScript setup** provides type safety and modern JavaScript features: 

```json
{
  "compilerOptions": {
    "target": "ES2018",
    "lib": ["DOM", "DOM.Iterable", "ES2018"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "target"]
}
```

### Novel-inspired AI editor component

The **core editor component** adapts Novel’s architecture for Confluence integration: 

```typescript
import React, { useState, useEffect } from 'react';
import { Editor, EditorContent, useEditor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import { AIExtension } from './extensions/AIExtension';
import { SlashCommand } from './extensions/SlashCommand';
import { BubbleMenu } from './components/BubbleMenu';

interface AIEditorProps {
    defaultValue?: string;
    onUpdate?: (content: string) => void;
    spaceKey?: string;
    pageId?: string;
}

export const AIEditor: React.FC<AIEditorProps> = ({ 
    defaultValue, 
    onUpdate, 
    spaceKey, 
    pageId 
}) => {
    const [isAILoading, setIsAILoading] = useState(false);
    const [aiSuggestion, setAiSuggestion] = useState<string | null>(null);

    const editor = useEditor({
        extensions: [
            StarterKit,
            AIExtension.configure({
                onAIRequest: handleAIRequest,
                apiEndpoint: '/rest/ai/1.0/generate'
            }),
            SlashCommand.configure({
                suggestion: {
                    items: ({ query }) => {
                        return [
                            {
                                title: 'AI Continue',
                                description: 'Continue writing with AI',
                                command: ({ editor, range }) => {
                                    editor.chain().focus().deleteRange(range).run();
                                    triggerAI('continue');
                                }
                            },
                            {
                                title: 'AI Improve',
                                description: 'Improve selected text',
                                command: ({ editor, range }) => {
                                    editor.chain().focus().deleteRange(range).run();
                                    triggerAI('improve');
                                }
                            },
                            {
                                title: 'AI Summarize',
                                description: 'Summarize content',
                                command: ({ editor, range }) => {
                                    editor.chain().focus().deleteRange(range).run();
                                    triggerAI('summarize');
                                }
                            }
                        ].filter(item => 
                            item.title.toLowerCase().includes(query.toLowerCase())
                        );
                    }
                }
            })
        ],
        content: defaultValue || '',
        onUpdate: ({ editor }) => {
            const html = editor.getHTML();
            onUpdate?.(html);
        },
        editorProps: {
            attributes: {
                class: 'prose prose-sm sm:prose lg:prose-lg xl:prose-2xl mx-auto focus:outline-none'
            }
        }
    });

    const handleAIRequest = async (prompt: string, context: string) => {
        setIsAILoading(true);
        try {
            const response = await fetch('/rest/ai/1.0/generate', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-Atlassian-Token': 'no-check'
                },
                body: JSON.stringify({
                    prompt,
                    context,
                    spaceKey,
                    pageId,
                    model: 'gpt-3.5-turbo'
                })
            });

            if (!response.ok) {
                throw new Error('AI request failed');
            }

            const data = await response.json();
            return data.content;
        } catch (error) {
            console.error('AI request error:', error);
            throw error;
        } finally {
            setIsAILoading(false);
        }
    };

    const triggerAI = async (command: string) => {
        const selection = editor?.state.selection;
        const selectedText = editor?.state.doc.textBetween(
            selection?.from || 0, 
            selection?.to || 0
        );
        
        const context = editor?.getHTML() || '';
        
        let prompt = '';
        switch (command) {
            case 'continue':
                prompt = `Continue writing the following text: ${selectedText || context}`;
                break;
            case 'improve':
                prompt = `Improve the following text: ${selectedText}`;
                break;
            case 'summarize':
                prompt = `Summarize the following content: ${context}`;
                break;
        }

        try {
            const aiContent = await handleAIRequest(prompt, context);
            if (aiContent) {
                editor?.chain().focus().insertContent(aiContent).run();
            }
        } catch (error) {
            // Handle error appropriately
            console.error('AI generation failed:', error);
        }
    };

    if (!editor) {
        return <div>Loading editor...</div>;
    }

    return (
        <div className="ai-editor-container">
            <BubbleMenu editor={editor} />
            <EditorContent editor={editor} />
            {isAILoading && (
                <div className="ai-loading-indicator">
                    🤖 AI is thinking...
                </div>
            )}
        </div>
    );
};
```

### Frontend package.json

The **package configuration** includes all necessary dependencies for the Novel-style editor: 

```json
{
  "name": "ai-editor-frontend",
  "version": "1.0.0",
  "scripts": {
    "build": "webpack --mode=production",
    "dev": "webpack --mode=development --watch",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "@tiptap/core": "^2.1.0",
    "@tiptap/react": "^2.1.0",
    "@tiptap/starter-kit": "^2.1.0",
    "@tiptap/extension-bubble-menu": "^2.1.0",
    "@tiptap/suggestion": "^2.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@babel/core": "^7.22.0",
    "@babel/preset-env": "^7.22.0",
    "@babel/preset-react": "^7.22.0",
    "@babel/preset-typescript": "^7.22.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "atlassian-webresource-webpack-plugin": "^6.0.0",
    "babel-loader": "^9.1.0",
    "css-loader": "^6.8.0",
    "style-loader": "^3.3.0",
    "typescript": "^5.1.0",
    "webpack": "^5.88.0",
    "webpack-cli": "^5.1.0"
  }
}
```

## Backend Java implementation

### AI service integration

The **AI service** handles external API integration with proper security and error handling:

```java
package com.example.plugin.service;

import com.atlassian.plugin.spring.scanner.annotation.component.Scanned;
import com.atlassian.plugin.spring.scanner.annotation.imports.ComponentImport;
import com.atlassian.sal.api.pluginsettings.PluginSettingsFactory;
import org.springframework.stereotype.Component;

import javax.inject.Inject;
import java.io.IOException;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

@Scanned
@Component
public class AIService {
    
    private final PluginSettingsFactory pluginSettingsFactory;
    private final HttpClient httpClient;
    private final AIContentValidator contentValidator;
    
    @Inject
    public AIService(@ComponentImport PluginSettingsFactory pluginSettingsFactory) {
        this.pluginSettingsFactory = pluginSettingsFactory;
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();
        this.contentValidator = new AIContentValidator();
    }
    
    public CompletableFuture<String> generateContent(String prompt, String context, String model) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                // Validate and filter content for security
                String safePrompt = contentValidator.filterSensitiveContent(prompt);
                String safeContext = contentValidator.filterSensitiveContent(context);
                
                if (contentValidator.containsSensitiveData(prompt) || 
                    contentValidator.containsSensitiveData(context)) {
                    throw new SecurityException("Content contains sensitive data");
                }
                
                String apiKey = getApiKey();
                String requestBody = buildOpenAIRequest(safePrompt, safeContext, model);
                
                HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create("https://api.openai.com/v1/chat/completions"))
                    .header("Authorization", "Bearer " + apiKey)
                    .header("Content-Type", "application/json")
                    .timeout(Duration.ofSeconds(30))
                    .POST(HttpRequest.BodyPublishers.ofString(requestBody))
                    .build();
                
                HttpResponse<String> response = httpClient.send(request, 
                    HttpResponse.BodyHandlers.ofString());
                
                if (response.statusCode() != 200) {
                    throw new AIServiceException("AI API returned error: " + response.statusCode());
                }
                
                return parseOpenAIResponse(response.body());
                
            } catch (Exception e) {
                throw new AIServiceException("Failed to generate AI content", e);
            }
        });
    }
    
    private String getApiKey() {
        return (String) pluginSettingsFactory.createGlobalSettings()
            .get("ai.api.key");
    }
    
    private String buildOpenAIRequest(String prompt, String context, String model) {
        return String.format("""
            {
                "model": "%s",
                "messages": [
                    {
                        "role": "system",
                        "content": "You are a helpful writing assistant for Confluence. Help improve and generate content based on the context provided."
                    },
                    {
                        "role": "user", 
                        "content": "Context: %s\\n\\nPrompt: %s"
                    }
                ],
                "max_tokens": 1000,
                "temperature": 0.7
            }
            """, model, context.replace("\"", "\\\""), prompt.replace("\"", "\\\""));
    }
    
    private String parseOpenAIResponse(String responseBody) {
        // Parse JSON response and extract content
        // Implement proper JSON parsing here
        return "AI generated content"; // Placeholder
    }
}
```

### REST controller for AI endpoints

The **REST controller** provides secure API endpoints for frontend integration:

```java
package com.example.plugin.rest;

import com.atlassian.plugins.rest.common.security.AnonymousAllowed;
import com.atlassian.sal.api.user.UserManager;
import com.example.plugin.service.AIService;

import javax.inject.Inject;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import java.util.concurrent.CompletableFuture;

@Path("/")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class AIController {
    
    private final AIService aiService;
    private final UserManager userManager;
    
    @Inject
    public AIController(AIService aiService, UserManager userManager) {
        this.aiService = aiService;
        this.userManager = userManager;
    }
    
    @POST
    @Path("/generate")
    public Response generateContent(AIRequest request) {
        try {
            // Validate user permissions
            if (!hasPermission(request.getSpaceKey())) {
                return Response.status(Response.Status.FORBIDDEN)
                    .entity("Insufficient permissions")
                    .build();
            }
            
            CompletableFuture<String> contentFuture = aiService.generateContent(
                request.getPrompt(),
                request.getContext(),
                request.getModel()
            );
            
            String content = contentFuture.join();
            
            return Response.ok(new AIResponse(content, "success")).build();
            
        } catch (Exception e) {
            return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
                .entity(new AIResponse(null, "Error: " + e.getMessage()))
                .build();
        }
    }
    
    @GET
    @Path("/models")
    public Response getAvailableModels() {
        String[] models = {"gpt-3.5-turbo", "gpt-4"};
        return Response.ok(models).build();
    }
    
    private boolean hasPermission(String spaceKey) {
        // Implement proper permission checking
        return userManager.getRemoteUser() != null;
    }
    
    // Request/Response DTOs
    public static class AIRequest {
        private String prompt;
        private String context;
        private String model;
        private String spaceKey;
        private String pageId;
        
        // Getters and setters
        public String getPrompt() { return prompt; }
        public void setPrompt(String prompt) { this.prompt = prompt; }
        // ... other getters/setters
    }
    
    public static class AIResponse {
        private String content;
        private String status;
        
        public AIResponse(String content, String status) {
            this.content = content;
            this.status = status;
        }
        
        // Getters and setters
        public String getContent() { return content; }
        public String getStatus() { return status; }
    }
}
```

### Confluence macro implementation

The **macro component** embeds the AI editor into Confluence pages: 

```java
package com.example.plugin.macro;

import com.atlassian.confluence.content.render.xhtml.ConversionContext;
import com.atlassian.confluence.macro.Macro;
import com.atlassian.confluence.macro.MacroExecutionException;
import com.atlassian.confluence.util.velocity.VelocityUtils;
import com.atlassian.plugin.spring.scanner.annotation.component.Scanned;
import com.atlassian.plugin.spring.scanner.annotation.imports.ComponentImport;
import com.atlassian.webresource.api.assembler.PageBuilderService;

import javax.inject.Inject;
import java.util.Map;

@Scanned
public class AIEditorMacro implements Macro {
    
    private final PageBuilderService pageBuilderService;
    
    @Inject
    public AIEditorMacro(@ComponentImport PageBuilderService pageBuilderService) {
        this.pageBuilderService = pageBuilderService;
    }
    
    @Override
    public String execute(Map<String, String> parameters, String body, ConversionContext conversionContext) 
            throws MacroExecutionException {
        
        // Include required web resources
        pageBuilderService.assembler().resources()
            .requireWebResource("com.example.confluence.ai-editor-plugin:ai-editor-resources");
        
        // Prepare template context
        Map<String, Object> context = Map.of(
            "model", parameters.getOrDefault("model", "gpt-3.5-turbo"),
            "prompt", parameters.getOrDefault("prompt", ""),
            "spaceKey", conversionContext.getSpaceKey(),
            "pageId", conversionContext.getEntity().getId().toString(),
            "body", body != null ? body : ""
        );
        
        return VelocityUtils.getRenderedTemplate("templates/ai-editor-macro.vm", context);
    }
    
    @Override
    public BodyType getBodyType() {
        return BodyType.RICH_TEXT;
    }
    
    @Override
    public OutputType getOutputType() {
        return OutputType.BLOCK;
    }
}
```

## AI integration implementation

### Security-first AI integration

**Content filtering and validation** prevents sensitive data exposure:  

```java
package com.example.plugin.service;

import java.util.Arrays;
import java.util.List;
import java.util.regex.Pattern;

public class AIContentValidator {
    
    private final List<Pattern> sensitivePatterns = Arrays.asList(
        Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b"), // SSN
        Pattern.compile("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"), // Credit cards
        Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b"), // Email
        Pattern.compile("\\b(?:password|pwd|secret|token|key)\\s*[:=]\\s*\\S+", Pattern.CASE_INSENSITIVE)
    );
    
    public String filterSensitiveContent(String content) {
        String filtered = content;
        for (Pattern pattern : sensitivePatterns) {
            filtered = pattern.matcher(filtered).replaceAll("[REDACTED]");
        }
        return filtered;
    }
    
    public boolean containsSensitiveData(String content) {
        return sensitivePatterns.stream()
            .anyMatch(pattern -> pattern.matcher(content).find());
    }
}
```

### Performance optimization with caching

**Intelligent caching** reduces API costs and improves response times:  

```java
package com.example.plugin.service;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class AIResponseCache {
    
    private final ConcurrentMap<String, CachedResponse> cache = new ConcurrentHashMap<>();
    private final long ttlMillis = 3600000; // 1 hour
    private final int maxSize = 1000;
    
    public String get(String promptHash) {
        CachedResponse cached = cache.get(promptHash);
        if (cached == null || isExpired(cached)) {
            cache.remove(promptHash);
            return null;
        }
        return cached.response;
    }
    
    public void put(String promptHash, String response) {
        if (cache.size() >= maxSize) {
            evictOldest();
        }
        cache.put(promptHash, new CachedResponse(response, System.currentTimeMillis()));
    }
    
    private boolean isExpired(CachedResponse cached) {
        return System.currentTimeMillis() - cached.timestamp > ttlMillis;
    }
    
    private void evictOldest() {
        String oldestKey = cache.entrySet().stream()
            .min((e1, e2) -> Long.compare(e1.getValue().timestamp, e2.getValue().timestamp))
            .map(Map.Entry::getKey)
            .orElse(null);
        
        if (oldestKey != null) {
            cache.remove(oldestKey);
        }
    }
    
    private static class CachedResponse {
        final String response;
        final long timestamp;
        
        CachedResponse(String response, long timestamp) {
            this.response = response;
            this.timestamp = timestamp;
        }
    }
}
```

## Development workflow and deployment

### Local development setup

Use the **Atlassian Plugin SDK** for streamlined development:

```bash
# Create project structure
atlas-create-confluence-plugin --artifact-id ai-editor-plugin --group-id com.example.confluence

# Start local Confluence with plugin
atlas-run

# Debug mode for development
atlas-debug

# Package for deployment
atlas-package
```

### Testing procedures for Data Center environments

**Comprehensive testing** ensures Data Center compatibility:

```bash
# Unit tests
mvn test

# Integration tests with test data
atlas-integration-test

# Data Center compatibility validation
java -jar cdc-plugin-validator-1.0.0.jar \
    -installation /path/to/confluence \
    -dbuser confuser -dbpassword confuser \
    -dburl jdbc:postgresql://localhost:5432/confluence
```

**Performance testing** with realistic workloads:

- Use Data Center App Performance Toolkit for load testing
- Test with large datasets and concurrent users
- Verify cluster behavior with multiple nodes
- Monitor memory usage and response times

### Deployment best practices

**Production deployment** follows these steps:

1. **Build and package**: `atlas-package` creates the deployable JAR
1. **Security review**: Validate all AI integrations and data handling
1. **Performance testing**: Confirm acceptable response times under load
1. **Staged rollout**: Deploy to test environments first
1. **Monitoring setup**: Configure logging and metrics collection

**Data Center specific considerations**:

- Plugin JARs are automatically distributed across cluster nodes
- Shared home directory must be accessible to all nodes
- Configuration changes require plugin restart on all nodes
- Monitor cluster messaging for AI operations

## Security and enterprise considerations

### Data protection strategies

**Enterprise-grade security** protects sensitive information:

- **Data classification**: Implement content scanning before AI processing
- **Encryption**: Use TLS 1.3 for all API communications
- **Access control**: Integrate with Confluence’s permission system
- **Audit logging**: Track all AI interactions for compliance
- **Data residency**: Consider regional AI service deployment

### Performance monitoring

**Observability** ensures optimal operation:

```java
@Component
public class AIMetrics {
    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final Timer responseTimer;
    
    public void recordAIRequest(String operation, String model) {
        requestCounter.increment(
            Tags.of("operation", operation, "model", model)
        );
    }
    
    public void recordResponseTime(String operation, Duration duration) {
        responseTimer.record(duration, TimeUnit.MILLISECONDS);
    }
}
```

## Complete implementation roadmap

This comprehensive guide provides everything needed to create a **production-ready Confluence Data Center 8.5 plugin** with AI-powered editing capabilities. The implementation combines:

**Modern frontend architecture** with webpack bundling, TypeScript support, and React components inspired by Novel’s design patterns. The Tiptap-based editor provides extensible rich text editing with AI integration points.

**Robust backend services** handle AI API integration with proper security, caching, and error handling. The REST API provides secure endpoints for frontend communication while maintaining Confluence’s permission model.

**Enterprise-ready deployment** supports Data Center clustering requirements with stateless design, proper resource management, and comprehensive testing procedures.

**Security-first approach** implements content filtering, sensitive data detection, and audit logging to meet enterprise compliance requirements while enabling powerful AI capabilities.

The resulting plugin delivers a **Novel-like editing experience** within Confluence’s enterprise environment, providing users with AI-assisted content creation while maintaining the security and scalability requirements of Data Center deployments.


# Complete Confluence AI Editor Plugin Directory Structure

```
ai-editor-confluence-plugin/
├── pom.xml                                    # Main Maven configuration
├── README.md                                  # Project documentation
├── .gitignore                                 # Git ignore patterns
├── CHANGELOG.md                               # Version history
├── LICENSE                                    # License file
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── plugin/
│   │   │               ├── macro/
│   │   │               │   └── AIEditorMacro.java         # Confluence macro implementation
│   │   │               ├── rest/
│   │   │               │   ├── AIController.java          # REST API endpoints
│   │   │               │   ├── dto/
│   │   │               │   │   ├── AIRequest.java         # Request DTOs
│   │   │               │   │   └── AIResponse.java        # Response DTOs
│   │   │               │   └── filter/
│   │   │               │       └── SecurityFilter.java   # Security filters
│   │   │               ├── service/
│   │   │               │   ├── AIService.java             # Main AI service
│   │   │               │   ├── AIContentValidator.java   # Content validation
│   │   │               │   ├── AIResponseCache.java      # Caching service
│   │   │               │   └── AIMetrics.java             # Metrics collection
│   │   │               ├── config/
│   │   │               │   ├── PluginConfiguration.java  # Plugin config
│   │   │               │   └── SecurityConfiguration.java # Security config
│   │   │               ├── exception/
│   │   │               │   ├── AIServiceException.java   # Custom exceptions
│   │   │               │   └── SecurityException.java    # Security exceptions
│   │   │               └── util/
│   │   │                   ├── JsonUtils.java            # JSON utilities
│   │   │                   └── HashUtils.java            # Hashing utilities
│   │   │
│   │   └── resources/
│   │       ├── atlassian-plugin.xml                      # Plugin descriptor
│   │       ├── templates/
│   │       │   ├── ai-editor-macro.vm                    # Velocity template for macro
│   │       │   └── admin/
│   │       │       └── configuration.vm                 # Admin configuration page
│   │       ├── i18n/
│   │       │   ├── ai-editor.properties                  # Default translations
│   │       │   ├── ai-editor_en.properties               # English translations
│   │       │   ├── ai-editor_es.properties               # Spanish translations
│   │       │   └── ai-editor_fr.properties               # French translations
│   │       ├── images/
│   │       │   ├── plugin-icon.png                       # Plugin icon
│   │       │   ├── ai-icon.svg                          # AI feature icon
│   │       │   └── editor-icons/
│   │       │       ├── continue.svg                      # Continue writing icon
│   │       │       ├── improve.svg                       # Improve text icon
│   │       │       └── summarize.svg                     # Summarize icon
│   │       ├── css/
│   │       │   └── ai-editor-admin.css                   # Admin panel styles
│   │       └── META-INF/
│   │           └── spring/
│   │               └── plugin-context.xml                # Spring configuration
│   │
│   └── test/
│       ├── java/
│       │   └── com/
│       │       └── example/
│       │           └── plugin/
│       │               ├── macro/
│       │               │   └── AIEditorMacroTest.java    # Macro tests
│       │               ├── rest/
│       │               │   └── AIControllerTest.java     # REST API tests
│       │               ├── service/
│       │               │   ├── AIServiceTest.java        # Service tests
│       │               │   ├── AIContentValidatorTest.java # Validator tests
│       │               │   └── AIResponseCacheTest.java  # Cache tests
│       │               └── integration/
│       │                   └── AIIntegrationTest.java    # Integration tests
│       │
│       └── resources/
│           ├── test-data/
│           │   ├── sample-content.html                   # Test content
│           │   └── ai-responses.json                     # Mock AI responses
│           └── logback-test.xml                          # Test logging config
│
├── frontend/                                             # Frontend React/TypeScript code
│   ├── package.json                                      # NPM configuration
│   ├── package-lock.json                                 # NPM lock file
│   ├── webpack.config.js                                 # Webpack configuration
│   ├── webpack.prod.js                                   # Production webpack config
│   ├── webpack.dev.js                                    # Development webpack config
│   ├── tsconfig.json                                     # TypeScript configuration
│   ├── .eslintrc.js                                      # ESLint configuration
│   ├── .prettierrc                                       # Prettier configuration
│   │
│   ├── src/
│   │   ├── index.tsx                                     # Main entry point
│   │   ├── types/
│   │   │   ├── ai.types.ts                              # AI-related types
│   │   │   ├── editor.types.ts                          # Editor types
│   │   │   └── confluence.types.ts                      # Confluence types
│   │   │
│   │   ├── components/
│   │   │   ├── AIEditor.tsx                             # Main AI editor component
│   │   │   ├── NovelEditor.tsx                          # Novel-style editor
│   │   │   ├── AIAssistant.tsx                          # AI assistant panel
│   │   │   ├── BubbleMenu.tsx                           # Floating menu
│   │   │   ├── SlashMenu.tsx                            # Slash command menu
│   │   │   ├── LoadingSpinner.tsx                       # Loading indicator
│   │   │   └── ErrorBoundary.tsx                        # Error handling
│   │   │
│   │   ├── extensions/
│   │   │   ├── AIExtension.ts                           # TipTap AI extension
│   │   │   ├── SlashCommand.ts                          # Slash command extension
│   │   │   ├── BubbleMenuExtension.ts                   # Bubble menu extension
│   │   │   └── HighlightExtension.ts                    # Text highlighting
│   │   │
│   │   ├── services/
│   │   │   ├── AIService.ts                             # Frontend AI service
│   │   │   ├── ConfluenceAPI.ts                         # Confluence API client
│   │   │   ├── StorageService.ts                        # Local storage service
│   │   │   └── EventService.ts                          # Event management
│   │   │
│   │   ├── hooks/
│   │   │   ├── useAI.ts                                 # AI operations hook
│   │   │   ├── useEditor.ts                             # Editor state hook
│   │   │   ├── useDebounce.ts                           # Debouncing hook
│   │   │   └── useLocalStorage.ts                       # Storage hook
│   │   │
│   │   ├── utils/
│   │   │   ├── api.utils.ts                             # API utilities
│   │   │   ├── text.utils.ts                            # Text processing
│   │   │   ├── validation.utils.ts                      # Input validation
│   │   │   └── constants.ts                             # App constants
│   │   │
│   │   └── styles/
│   │       ├── editor.css                               # Editor styles
│   │       ├── ai-assistant.css                         # AI assistant styles
│   │       ├── bubble-menu.css                          # Bubble menu styles
│   │       ├── slash-menu.css                           # Slash menu styles
│   │       ├── variables.css                            # CSS variables
│   │       └── animations.css                           # Animation styles
│   │
│   ├── public/
│   │   └── icons/
│   │       ├── ai.svg                                   # AI icon
│   │       ├── magic-wand.svg                           # Magic wand icon
│   │       └── sparkles.svg                             # Sparkles icon
│   │
│   └── dist/                                            # Build output (generated)
│       ├── ai-editor.js                                 # Bundled JavaScript
│       ├── ai-editor.css                                # Bundled CSS
│       └── assets/                                      # Static assets
│
├── docs/                                                # Documentation
│   ├── installation.md                                  # Installation guide
│   ├── configuration.md                                 # Configuration guide
│   ├── api.md                                          # API documentation
│   ├── troubleshooting.md                              # Troubleshooting guide
│   └── screenshots/                                     # Feature screenshots
│       ├── editor-interface.png
│       ├── ai-suggestions.png
│       └── admin-panel.png
│
├── scripts/                                            # Build and deployment scripts
│   ├── build.sh                                        # Build script
│   ├── deploy.sh                                       # Deployment script
│   ├── test.sh                                         # Test runner script
│   └── release.sh                                      # Release script
│
├── config/                                             # Configuration files
│   ├── checkstyle.xml                                  # Code style rules
│   ├── spotbugs-exclude.xml                            # SpotBugs exclusions
│   └── sonar-project.properties                       # SonarQube config
│
├── target/                                             # Maven build output (generated)
│   ├── classes/
│   │   ├── com/example/plugin/                         # Compiled Java classes
│   │   ├── templates/                                  # Processed templates
│   │   ├── i18n/                                       # Processed translations
│   │   ├── assets/                                     # Frontend build output
│   │   │   ├── ai-editor.js                            # Bundled JS
│   │   │   └── ai-editor.css                           # Bundled CSS
│   │   └── META-INF/
│   │       ├── MANIFEST.MF                             # JAR manifest
│   │       └── plugin-descriptors/
│   │           └── wr-webpack-bundles.xml              # Webpack web resources
│   ├── test-classes/                                   # Test classes
│   ├── surefire-reports/                               # Test reports
│   ├── ai-editor-plugin-1.0.0-SNAPSHOT.jar            # Final plugin JAR
│   └── ai-editor-plugin-1.0.0-SNAPSHOT-tests.jar      # Test JAR
│
├── .github/                                            # GitHub Actions (optional)
│   └── workflows/
│       ├── ci.yml                                      # Continuous integration
│       ├── release.yml                                 # Release workflow
│       └── security.yml                                # Security scanning
│
├── docker/                                             # Docker setup (optional)
│   ├── Dockerfile                                      # Plugin container
│   ├── docker-compose.yml                              # Local development
│   └── confluence/
│       └── setenv.sh                                   # Confluence environment
│
└── node_modules/                                       # NPM dependencies (generated)
    └── ...                                             # Frontend dependencies
```

## Key File Descriptions

### Root Level Files

- **pom.xml**: Main Maven configuration orchestrating frontend and backend builds
- **README.md**: Comprehensive setup and usage documentation
- **.gitignore**: Excludes build artifacts, dependencies, IDE files

### Java Backend (`src/main/java/`)

- **macro/AIEditorMacro.java**: Confluence macro that embeds the AI editor
- **rest/AIController.java**: REST endpoints for AI operations
- **service/AIService.java**: Core AI integration service with caching and validation
- **config/**: Spring configuration and plugin setup

### Resources (`src/main/resources/`)

- **atlassian-plugin.xml**: Plugin descriptor defining all components and dependencies
- **templates/**: Velocity templates for macro rendering
- **i18n/**: Internationalization files for multiple languages

### Frontend (`frontend/src/`)

- **components/AIEditor.tsx**: Main editor component with Novel-style interface
- **extensions/**: TipTap extensions for AI functionality and slash commands
- **services/**: API clients and utility services
- **styles/**: CSS modules for component styling

### Build Configuration

- **webpack.config.js**: Bundles frontend code with Atlassian web resource integration
- **tsconfig.json**: TypeScript compilation settings
- **package.json**: NPM dependencies and build scripts

### Testing Structure

- **src/test/java/**: Unit and integration tests for all Java components
- **frontend/src/**/*.test.ts**: Frontend component and service tests

This structure follows Maven conventions while supporting modern frontend development with proper separation of concerns and enterprise-grade organization.
