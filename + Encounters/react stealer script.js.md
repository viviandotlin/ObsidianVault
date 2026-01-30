```json
const ReactStealer = (function () {
    function findReactFiberKey(element) {
        const keys = Object.keys(element);
        return keys.find(key =>
            key.startsWith('__reactFiber$') ||
            key.startsWith('__reactInternalInstance$')
        );
    }

    function getComponentName(fiber) {
        if (!fiber?.type) return null;
        if (typeof fiber.type === 'string') return fiber.type;
        return fiber.type.displayName || fiber.type.name || 'Anonymous';
    }

    function isReactComponent(fiber) {
        return fiber?.type && typeof fiber.type === 'function';
    }

    function findDOMNode(fiber) {
        if (fiber.stateNode?.tagName) return fiber.stateNode;
        let child = fiber.child;
        let depth = 0;
        while (child && depth < 10) {
            if (child.stateNode?.tagName) return child.stateNode;
            child = child.child;
            depth++;
        }
        return null;
    }

    function getParentChain(fiber, maxDepth = 10) {
        const chain = [];
        let current = fiber?.return;
        let depth = 0;
        while (current && depth < maxDepth) {
            if (isReactComponent(current)) {
                chain.push({ name: getComponentName(current), hasState: !!current.memoizedState });
            }
            current = current.return;
            depth++;
        }
        return chain;
    }

    function extractHooks(fiber) {
        if (!fiber.memoizedState) return [];
        const hooks = [];
        let state = fiber.memoizedState;
        let idx = 0;
        while (state && idx < 30) {
            const hook = { index: idx };
            if (state.queue?.dispatch) {
                hook.type = 'useState';
                hook.valueType = typeof state.memoizedState;
            } else if (state.deps !== undefined) {
                hook.type = state.create ? 'useEffect' : 'useMemo/useCallback';
                hook.depsCount = state.deps?.length ?? 0;
            } else if (state.memoizedState?.current !== undefined) {
                hook.type = 'useRef';
            } else {
                hook.type = 'unknown';
                hook.valueType = typeof state.memoizedState;
            }
            hooks.push(hook);
            state = state.next;
            idx++;
        }
        return hooks;
    }

    function cleanProps(props) {
        if (!props) return {};
        const clean = {};
        for (const [key, value] of Object.entries(props)) {
            if (key === 'children') continue;
            if (typeof value === 'function') clean[key] = '[Function]';
            else if (value?.$$typeof) clean[key] = '[ReactElement]';
            else if (Array.isArray(value)) clean[key] = '[Array(' + value.length + ')]';
            else if (typeof value === 'object' && value !== null) {
                try { clean[key] = JSON.parse(JSON.stringify(value)); }
                catch { clean[key] = '[Object]'; }
            } else clean[key] = value;
        }
        return clean;
    }

    function extractAll(options = {}) {
        const {
            maxInstancesPerComponent = 5,
            includeSource = true,
            includeHTML = true,
            includeHooks = true,
            htmlMaxLength = 500,
            sourceMaxLength = 2000
        } = options;

        const componentMap = new Map();
        const seenFibers = new WeakSet();

        document.querySelectorAll('*').forEach(el => {
            const key = findReactFiberKey(el);
            if (!key) return;
            let fiber = el[key];
            while (fiber) {
                if (isReactComponent(fiber) && !seenFibers.has(fiber)) {
                    seenFibers.add(fiber);
                    const type = fiber.type;
                    const name = getComponentName(fiber);
                    if (!componentMap.has(type)) {
                        componentMap.set(type, { name, type, instances: [] });
                    }
                    const entry = componentMap.get(type);
                    if (entry.instances.length < maxInstancesPerComponent) {
                        const domNode = findDOMNode(fiber);
                        const instance = {
                            props: cleanProps(fiber.memoizedProps),
                            parentChain: getParentChain(fiber).map(p => p.name)
                        };
                        if (includeHooks) instance.hooks = extractHooks(fiber);
                        if (includeHTML && domNode) {
                            instance.renderedHTML = domNode.outerHTML.substring(0, htmlMaxLength);
                            instance.css = dumpCSSText(domNode)
                            instance.domTag = domNode.tagName;
                        }
                        entry.instances.push(instance);
                    }
                }
                fiber = fiber.return;
            }
        });

        return Array.from(componentMap.entries()).map(([type, data]) => {
            const result = { name: data.name, instanceCount: data.instances.length, instances: data.instances };
            if (includeSource && typeof type === 'function') {
                try { result.source = type.toString().substring(0, sourceMaxLength); }
                catch { result.source = null; }
            }
            return result;
        }).sort((a, b) => b.instanceCount - a.instanceCount);
    }

    function getBySelector(selector) {
        const el = document.querySelector(selector);
        if (!el) return null;
        const key = findReactFiberKey(el);
        if (!key) return null;
        let fiber = el[key];
        while (fiber && !isReactComponent(fiber)) fiber = fiber.return;
        if (!fiber) return null;
        const domNode = findDOMNode(fiber);
        return {
            name: getComponentName(fiber),
            props: cleanProps(fiber.memoizedProps),
            hooks: extractHooks(fiber),
            parentChain: getParentChain(fiber),
            renderedHTML: domNode?.outerHTML ?? null,
            source: typeof fiber.type === 'function' ? fiber.type.toString() : null
        };
    }

    function dumpCSSText(element) {
        var s = '';
        var o = getComputedStyle(element);
        for (var i = 0; i < o.length; i++) {
            s += o[i] + ':' + o.getPropertyValue(o[i]) + ';';
        }
        return s;
    }

    function summary() {
        const components = extractAll({ includeSource: false, includeHTML: false, includeHooks: false });
        return { totalComponents: components.length, components: components.map(c => ({ name: c.name, count: c.instanceCount })) };
    }

    function mergeCSSProperties(cssStrings) {
        const cssMap = new Map();
        cssStrings.forEach(cssString => {
            if (!cssString) return;
            const declarations = cssString.split(';').filter(Boolean);
            declarations.forEach(decl => {
                const colonIndex = decl.indexOf(':');
                if (colonIndex === -1) return;
                const prop = decl.substring(0, colonIndex).trim();
                const value = decl.substring(colonIndex + 1).trim();
                if (prop && value) {
                    cssMap.set(prop, value);
                }
            });
        });
        return Array.from(cssMap.entries())
            .map(([prop, value]) => `${prop}: ${value};`)
            .join('\n');
    }

    function getForLLM(componentName) {
        const all = extractAll({ maxInstancesPerComponent: 10, htmlMaxLength: 1000, sourceMaxLength: 5000 });
        const component = all.find(c => c.name === componentName);

        if (!component) return null;

        let prompt = `## Component: ${component.name}\n\n`;
        prompt += `### Minified Source Code\n\`\`\`javascript\n${component.source}\n\`\`\`\n\n`;
        prompt += `### Examples (${component.instances.length} instances found)\n\n`;

        const allCSSStrings = [];

        component.instances.forEach((inst, i) => {
            prompt += `#### Example ${i + 1}\n`;
            prompt += `**Props:**\n\`\`\`json\n${JSON.stringify(inst.props, null, 2)}\n\`\`\`\n`;
            prompt += `**Rendered HTML:**\n\`\`\`html\n${inst.renderedHTML}\n\`\`\`\n`;
            if (inst.hooks?.length) {
                prompt += `**Hooks:** ${inst.hooks.map(h => h.type).join(', ')}\n`;
            }
            prompt += `**Parent Chain:** ${inst.parentChain.join(' > ')}\n\n`;
            if (inst.css) {
                allCSSStrings.push(inst.css);
            }
        });

        const mergedCSS = mergeCSSProperties(allCSSStrings);
        if (mergedCSS) {
            prompt += `### CSS\n\`\`\`css\n${mergedCSS}\n\`\`\`\n`;
        }

        return prompt;
    }

    return { extractAll, getBySelector, summary, getForLLM, findReactFiberKey };
})();

window.ReactStealer = ReactStealer;

```
