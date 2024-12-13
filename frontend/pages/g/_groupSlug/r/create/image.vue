<template>
  <div>
    <v-form ref="domUrlForm" @submit.prevent="createRecipe">
      <div>
        <v-card-title class="headline"> {{ $t('recipe.create-recipe-from-an-image') }} </v-card-title>
        <v-card-text>
          <p>{{ $t('recipe.create-recipe-from-an-image-description') }}</p>
          <v-container class="pa-0">
            <v-row>
              <v-col cols="auto" align-self="center">
                <AppButtonUpload
                  v-if="uploadedImages.length === 0"
                  class="ml-auto"
                  url="none"
                  file-name="image"
                  accept="image/*"
                  :text="$i18n.tc('recipe.upload-image')"
                  :text-btn="false"
                  :post="false"
                  :multiple="true"
                  @uploaded="uploadImages"
                />
              </v-col>
              <v-spacer />
            </v-row>

            <div v-if="uploadedImages.length > 0" class="mt-3">
              <v-row>
                <v-col v-for="(image, index) in uploadedImages" :key="index" cols="12" md="6">
                  <v-card>
                    <ImageCropper
                      :img="image.previewUrl"
                      cropper-height="200px"
                      cropper-width="100%"
                      @save="(blob) => updateUploadedImage(index, blob)"
                    />
                    <v-card-actions>
                      <v-btn color="error" @click="removeImage(index)">
                        <v-icon left>{{ $globals.icons.close }}</v-icon>
                        {{ $i18n.tc('recipe.remove-image') }}
                      </v-btn>
                    </v-card-actions>
                  </v-card>
                </v-col>
              </v-row>
            </div>
          </v-container>
        </v-card-text>
        <v-card-actions v-if="uploadedImages.length > 0">
          <div>
            <p style="width: 250px">
              <BaseButton rounded block type="submit" :loading="loading" />
            </p>
            <p>
              <v-checkbox
                v-model="shouldTranslate"
                hide-details
                :label="$t('recipe.should-translate-description')"
                :disabled="loading"
              />
            </p>
            <p v-if="loading" class="mb-0">
              {{ $t('recipe.please-wait-image-procesing') }}
            </p>
          </div>
        </v-card-actions>
      </div>
    </v-form>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, reactive, ref, toRefs, useContext, useRoute, useRouter } from "@nuxtjs/composition-api";
import { useUserApi } from "~/composables/api";
import { alert } from "~/composables/use-toast";

interface UploadedImage {
  file: File | Blob;
  name: string;
  previewUrl: string;
}

export default defineComponent({
  setup() {
    const state = reactive({
      loading: false,
    });

    const { i18n } = useContext();
    const api = useUserApi();
    const route = useRoute();
    const router = useRouter();
    const groupSlug = computed(() => route.value.params.groupSlug || "");

    const uploadedImages = ref<UploadedImage[]>([]);
    const shouldTranslate = ref(true);

    function uploadImages(files: File[]) {
      files.forEach(file => {
        uploadedImages.value.push({
          file,
          name: file.name,
          previewUrl: URL.createObjectURL(file)
        });
      });
    }

    function updateUploadedImage(index: number, blob: Blob) {
      if (uploadedImages.value[index]) {
        uploadedImages.value[index].file = blob;
        uploadedImages.value[index].previewUrl = URL.createObjectURL(blob);
      }
    }

    function removeImage(index: number) {
      uploadedImages.value.splice(index, 1);
    }

    async function createRecipe() {
      if (uploadedImages.value.length === 0) {
        return;
      }

      state.loading = true;
      const translateLanguage = shouldTranslate.value ? i18n.locale : undefined;

      const formData = new FormData();
      uploadedImages.value.forEach((image, index) => {
        formData.append('images', image.file, image.name);
      });

      if (translateLanguage) {
        formData.append('translateLanguage', translateLanguage);
      }

      try {
        const { data, error } = await api.recipes.createFromImages(formData);
        if (error || !data) {
          alert.error(i18n.tc("events.something-went-wrong"));
        } else {
          router.push(`/g/${groupSlug.value}/r/${data}`);
        }
      } catch (err) {
        alert.error(i18n.tc("events.something-went-wrong"));
      } finally {
        state.loading = false;
      }
    }

    return {
      ...toRefs(state),
      uploadedImages,
      shouldTranslate,
      uploadImages,
      updateUploadedImage,
      removeImage,
      createRecipe,
    };
  },
});
</script>
