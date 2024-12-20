<template>
  <EditorToolbarBox
    :text="$t('toolbar.rewriter.title')"
    icon="i-material-symbols:auto-fix"
  >
    <div class="space-y-4">
      <UiAlert v-if="!isAIAvailable" variant="warning">
        <p class="text-sm">
          AI Rewriter requires Chrome Canary with specific flags enabled. Please:
          <ol class="list-decimal ml-4 mt-2">
            <li>Use Chrome Canary (version ≥ 129.0.6639.0)</li>
            <li>Go to chrome://flags</li>
            <li>Enable #optimization-guide-on-device-model</li>
            <li>Enable #writer-api-for-gemini-nano</li>
            <li>Enable #rewriter-api-for-gemini-nano</li>
            <li>Restart Chrome</li>
          </ol>
        </p>
      </UiAlert>

      <div v-else class="space-y-2">
        <div class="flex flex-col gap-2">
          <UiInput
            v-model="textToRewrite"
            type="textarea"
            rows="3"
            :placeholder="$t('toolbar.rewriter.placeholder')"
            class="w-full min-h-[80px] resize-y"
            @input="logInput"
          />
          <div class="flex gap-2">
            <UiButton 
              @click="captureSelectedText" 
              size="sm"
              class="flex-shrink-0"
            >
              C Selected Text
            </UiButton>
            <UiButton 
              @click="rewrite" 
              size="sm"
              :disabled="isLoading || !textToRewrite"
              class="flex-shrink-0"
            >
              <span class="i-material-symbols:auto-fix mr-1" />
              {{ $t("toolbar.rewriter.btn") }}
            </UiButton>
          </div>
        </div>
         <!-- Add Note Here -->
         <p class="text-xs text-muted">
          {{ $t('toolbar.rewriter.select.note') }}
        </p>
        
        <UiDropdownMenu>
          <UiDropdownMenuTrigger asChild>
            <UiButton variant="outline" class="w-full justify-start">
              {{ tone }}
            </UiButton>
          </UiDropdownMenuTrigger>
          <UiDropdownMenuContent class="w-56">
            <UiDropdownMenuRadioGroup v-model="tone" @update:modelValue="logToneChange">
              <UiDropdownMenuRadioItem v-for="toneOption in toneOptions" 
                :key="toneOption"
                :value="toneOption"
              >
                {{ toneOption }}
              </UiDropdownMenuRadioItem>
            </UiDropdownMenuRadioGroup>
          </UiDropdownMenuContent>
        </UiDropdownMenu>

        <div v-if="parsedOptions.length" class="mt-4 space-y-4">
          <p class="text-sm font-medium">{{ $t('toolbar.rewriter.options') }}</p>
          <div class="space-y-3">
            <div 
              v-for="(option, index) in parsedOptions"
              :key="index"
              class="p-3 bg-muted rounded-md hover:bg-muted/80 transition-colors cursor-pointer"
              @click="copyToClipboard(option.text)"
            >
              <div class="flex justify-between items-center mb-1">
                <p class="text-sm font-medium text-primary">{{ option.title }}</p>
                <UiButton variant="ghost" size="sm" @click.stop="copyToClipboard(option.text)">
                  <span class="i-lucide-copy size-4" />
                </UiButton>
              </div>
              <p class="text-sm">{{ option.text }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </EditorToolbarBox>
</template>

<script lang="ts" setup>
import { ref, computed, onMounted } from 'vue';
import { useToast } from 'vue-toastification';
import type { Tone } from '@/types';

interface RewriteOption {
  title: string;
  text: string;
}

const toast = useToast();
const toneOptions = ['as-is', 'more-formal', 'more-casual'] as const;
const tone = ref<Tone>('as-is');
const isLoading = ref(false);
const textToRewrite = ref('');
const rewrittenText = ref('');
const isAIAvailable = ref(false);

const parsedOptions = computed<RewriteOption[]>(() => {
  if (!rewrittenText.value) return [];
  
  // Match both "**Option X" and "**Option X (Type):" patterns
  const options = rewrittenText.value.match(/\*\*Option \d+(?:\s*\([^)]+\))?:[\s\S]*?(?=\*\*Option|\s*$)/g) || [];
  
  return options.map(option => {
    // Extract the option number and type (if present)
    const titleMatch = option.match(/\*\*Option (\d+)(?:\s*\(([^)]+)\))?:/);
    const optionNumber = titleMatch?.[1] || '';
    const optionType = titleMatch?.[2] || '';
    
    // Clean up the text content
    const text = option
      .replace(/\*\*Option \d+(?:\s*\([^)]+\))?:/, '') // Remove the header
      .replace(/[>*]/g, '') // Remove markdown symbols
      .trim();

    // Construct the title
    const title = optionType 
      ? `Option ${optionNumber} (${optionType})`
      : `Option ${optionNumber}`;
    
    return {
      title,
      text
    };
  });
});

const checkAIAvailability = async (): Promise<boolean> => {
  if (typeof window === 'undefined' || !window.ai?.languageModel) {
    console.error('AI Language Model API not available');
    return false;
  }

  try {
    const capabilities = await window.ai.languageModel.capabilities();
    return capabilities.available !== 'no';
  } catch (error) {
    console.error('Error checking AI capabilities:', error);
    return false;
  }
};

const copyToClipboard = async (text: string) => {
  try {
    await navigator.clipboard.writeText(text);
    toast.success($t('toolbar.rewriter.copy.success'));
  } catch (error) {
    toast.error($t('toolbar.rewriter.copy.error'));
  }
};

const logInput = () => {
  console.log('Input Text Changed:', {
    text: textToRewrite.value,
    length: textToRewrite.value.length
  });
};

const logToneChange = (newTone: Tone) => {
  console.log('Tone Changed:', {
    from: tone.value,
    to: newTone
  });
};

const captureSelectedText = () => {
  // Get the currently selected text from the window
  const selectedText = window.getSelection()?.toString().trim();
  
  if (selectedText) {
    console.log('Selected Text:', selectedText);
    textToRewrite.value = selectedText;
    toast.success($t('toolbar.rewriter.select.success'));
    return true;
  } else {
    toast.info($t('toolbar.rewriter.select.empty'));
    return false;
  }
};

const rewrite = async () => {
  if (isLoading.value) return;

  if (!textToRewrite.value) {
    // Try to get selected text first
    const hasSelectedText = captureSelectedText();
    if (!hasSelectedText) {
      // If no selected text, try clipboard as fallback
      try {
        const clipboardText = await navigator.clipboard.readText();
        if (clipboardText.trim()) {
          textToRewrite.value = clipboardText;
          console.log('Clipboard Text:', clipboardText);
        } else {
          toast.warning($t('toolbar.rewriter.errors.no_input'));
          return;
        }
      } catch (error) {
        toast.warning($t('toolbar.rewriter.errors.no_input'));
        return;
      }
    }
  }

  isLoading.value = true;
  let session;

  try {
    if (!await checkAIAvailability()) {
      throw new Error($t('toolbar.rewriter.errors.ai_unavailable'));
    }

    const toneMapping = {
      'more-formal': 'more formal',
      'more-casual': 'more casual',
      'as-is': 'similar in tone',
    };

    const systemPrompt = `You are an AI assistant that helps rewrite text. 
      When given text, provide 3 rewritten versions labeled as "**Option 1 (Formal):", "**Option 2 (Informal):", and "**Option 3 (Concise):" 
      Make Option 1 more formal, Option 2 more casual, and Option 3 shorter while maintaining the core meaning.
      Each option should be clearly separated and maintain any original markdown formatting.`;

    session = await window.ai.languageModel.create({ systemPrompt });
    const result = await session.prompt(`Please rewrite the following text:\n\n${textToRewrite.value}`);
    rewrittenText.value = result;

    toast.success($t('toolbar.rewriter.success'));
  } catch (error) {
    console.error('Rewrite failed:', error);
    toast.error(error instanceof Error ? error.message : $t('toolbar.rewriter.errors.unknown'));
  } finally {
    if (session) {
      await session.destroy();
    }
    isLoading.value = false;
  }
};

onMounted(async () => {
  isAIAvailable.value = await checkAIAvailability();
});
</script>

<style scoped>
.size-4 {
  width: 1rem;
  height: 1rem;
}

/* Add these styles for the textarea */
:deep(.ui-input) {
  min-height: 80px;
  resize: vertical;
  line-height: 1.5;
  padding: 0.5rem;
}
</style>