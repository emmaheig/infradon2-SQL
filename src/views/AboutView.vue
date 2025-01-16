<script lang="ts">
import { defineComponent } from 'vue'
import PouchDB from 'pouchdb'

// Types
interface Post {
  _id: string
  _rev?: string
  title: string
  content: string
  attributes: {
    creation_date: string
    modified: string
  }
}

// Utilitaires
const createTimestamp = () => new Date().toISOString()

export default defineComponent({
  name: 'PostsManager',
  data() {
    return {
      localDb: null as PouchDB.Database<Post> | null,
      remoteDb: null as PouchDB.Database<Post> | null,
      sync: null as PouchDB.Replication.Sync<Post> | null,
      posts: [] as Post[],
      newPost: {
        title: '',
        content: '',
      },
      isEditing: false,
      currentEditId: null as string | null,
      error: null as string | null,
      isLoading: false,
      pageSize: 10,
      currentPage: 1,
    }
  },

  computed: {
    sortedPosts() {
      return [...this.posts].sort(
        (a, b) =>
          new Date(b.attributes.creation_date).getTime() -
          new Date(a.attributes.creation_date).getTime(),
      )
    },

    paginatedPosts() {
      const start = (this.currentPage - 1) * this.pageSize
      return this.sortedPosts.slice(start, start + this.pageSize)
    },

    totalPages() {
      return Math.ceil(this.posts.length / this.pageSize)
    },
  },

  methods: {
    async initDatabases() {
      try {
        if (!this.localDb) {
          this.localDb = new PouchDB<Post>('posts')
        }

        if (!this.remoteDb) {
          this.remoteDb = new PouchDB('http://ecs:Cats1234@localhost:5984/test1', {
            skip_setup: true,
          })
        }

        // Vérifier la connexion
        await this.remoteDb.info()

        // Configuration de la synchronisation
        if (this.localDb && this.remoteDb) {
          this.sync = this.localDb
            .sync(this.remoteDb, {
              live: true,
              retry: true,
              back_off_function: (delay: number) => delay * 2,
            })
            .on('change', (change) => {
              console.log('Sync change:', change)
              this.fetchPosts()
            })
            .on('paused', () => {
              console.log('Sync paused')
            })
            .on('active', () => {
              console.log('Sync active')
            })
            .on('denied', (err) => {
              console.error('Sync denied:', err)
              this.error = 'Accès refusé à la base de données'
            })
            .on('error', (error) => {
              console.error('Sync error:', error)
              this.error = 'Erreur de synchronisation: ' + (error as Error).message
            })
        }

        await this.fetchPosts()
      } catch (error) {
        const err = error as Error
        console.error('Database initialization error:', err.message)
        this.error = `Erreur d'initialisation: ${err.message}`
      }
    },

    async fetchPosts() {
      if (!this.localDb) return

      this.isLoading = true
      try {
        const result = await this.localDb.allDocs({
          include_docs: true,
          attachments: true,
        })
        this.posts = result.rows.map((row) => row.doc as Post)
        this.error = null
      } catch (error) {
        const err = error as Error
        console.error('Fetch error:', err)
        this.error = 'Erreur lors de la récupération des posts: ' + err.message
      } finally {
        this.isLoading = false
      }
    },

    validatePost(post: Partial<Post>): boolean {
      if (!post.title?.trim() || !post.content?.trim()) {
        this.error = 'Le titre et le contenu sont requis'
        return false
      }
      if (post.title.length > 100) {
        this.error = 'Le titre ne doit pas dépasser 100 caractères'
        return false
      }
      if (post.content.length > 1000) {
        this.error = 'Le contenu ne doit pas dépasser 1000 caractères'
        return false
      }
      return true
    },

    async addPost() {
      if (!this.localDb || !this.newPost.title.trim() || !this.newPost.content.trim()) return

      const post: Omit<Post, '_rev'> = {
        _id: createTimestamp(),
        title: this.newPost.title.trim(),
        content: this.newPost.content.trim(),
        attributes: {
          creation_date: createTimestamp(),
          modified: createTimestamp(),
        },
      }

      if (!this.validatePost(post)) return

      try {
        await this.localDb.put(post)
        this.newPost.title = ''
        this.newPost.content = ''
        await this.fetchPosts()
        this.error = null
      } catch (error) {
        const err = error as Error
        console.error('Add error:', err)
        this.error = "Erreur lors de l'ajout du post: " + err.message
      }
    },

      createTimestamp() {
    return new Date().toISOString();
  },

    async deletePost(post: Post) {
      if (!this.localDb || !post._id) return

      this.isLoading = true
      try {
        await this.localDb.remove(post._id, post._rev as string)
        await this.fetchPosts()
        this.error = null
      } catch (error) {
        const err = error as Error
        console.error('Delete error:', err)
        this.error = 'Erreur lors de la suppression du post : ' + err.message
      } finally {
        this.isLoading = false
      }
    },

    async updatePost(partialPost: Partial<Post>) {
      console.log('updatePost')
      if (!this.localDb || !partialPost._id) return

      try {
        // Récupérer le document existant
        const existingDoc = await this.localDb.get(partialPost._id)

        // Créer le document mis à jour
        const updatedPost: Post = {
          ...existingDoc, // Garder toutes les propriétés existantes
          title: partialPost.title || existingDoc.title,
          content: partialPost.content || existingDoc.content,
          attributes: {
            ...existingDoc.attributes,
            modified: createTimestamp(), // Mettre à jour uniquement la date de modification
          },
        }

        if (!this.validatePost(updatedPost)) return
        console.log('updatePost on fait vraiment la mise a jour')

        await this.localDb.put(updatedPost)
        this.isEditing = false
        this.currentEditId = null
        this.newPost = { title: '', content: '' }
        await this.fetchPosts()
        this.error = null
      } catch (error) {
        const err = error as Error
        console.error('Update error:', err)
        this.error = 'Erreur lors de la mise à jour du post: ' + err.message
      }
    },

    startEditing(post: Post) {
      this.isEditing = true
      this.currentEditId = post._id
      this.newPost = {
        title: post.title,
        content: post.content,
      }
    },

    cancelEditing() {
      this.isEditing = false
      this.currentEditId = null
      this.newPost = {
        title: '',
        content: '',
      }
    },

    changePage(page: number) {
      if (page >= 1 && page <= this.totalPages) {
        this.currentPage = page
      }
    },
  },

  mounted() {
    this.initDatabases()
  },

  beforeUnmount() {
    if (this.sync) {
      this.sync.cancel()
    }
    if (this.localDb) {
      this.localDb.close()
    }
    if (this.remoteDb) {
      this.remoteDb.close()
    }
  },
})
</script>

<template>
  <div class="posts-manager">
    <!-- Error Alert -->
    <div v-if="error" class="error-alert" role="alert">
      {{ error }}
      <button @click="error = null" class="close-button">×</button>
    </div>

    <!-- Loading Indicator -->
    <div v-if="isLoading" class="loading-indicator">Chargement...</div>

    <div class="content-container">
      <!-- Form Section -->
      <div class="form-section">
        <h1 class="section-title">{{ isEditing ? 'Modifier le post' : 'Nouveau post' }}</h1>
        <form
          @submit.prevent="
            isEditing
              ? updatePost({
                  _id: currentEditId as string,
                  title: newPost.title,
                  content: newPost.content,
                  attributes: {
                    creation_date: createTimestamp(),
                    modified: createTimestamp(),
                  },
                })
              : addPost()
          "
          class="post-form"
        >
          <div class="form-group">
            <input
              v-model="newPost.title"
              placeholder="Titre du post"
              required
              maxlength="100"
              class="form-input"
            />
            <small class="character-count">{{ newPost.title.length }}/100</small>
          </div>
          <div class="form-group">
            <textarea
              v-model="newPost.content"
              placeholder="Contenu du post"
              required
              maxlength="1000"
              class="form-textarea"
            ></textarea>
            <small class="character-count">{{ newPost.content.length }}/1000</small>
          </div>
          <div class="button-group">
            <button type="submit" class="submit-button" :disabled="isLoading">
              {{ isEditing ? 'Mettre à jour' : 'Ajouter' }}
            </button>
            <button
              v-if="isEditing"
              type="button"
              @click="cancelEditing"
              class="cancel-button"
              :disabled="isLoading"
            >
              Annuler
            </button>
          </div>
        </form>
      </div>

      <!-- Posts List -->
      <div class="posts-section">
        <h2 class="section-title">Posts ({{ posts.length }})</h2>
        <div class="posts-scrollable">
          <div v-if="paginatedPosts.length" class="posts-grid">
            <div v-for="post in paginatedPosts" :key="post._id" class="post-card">
              <div class="post-card-header">
                <h3 class="post-title">{{ post.title }}</h3>
                <div class="post-actions">
                  <button
                    @click="startEditing(post)"
                    class="icon-button edit-button"
                    title="Modifier"
                    :disabled="isLoading"
                  >
                    ✏
                  </button>
                  <button
                    @click="deletePost(post)"
                    class="icon-button delete-button"
                    title="Supprimer"
                    :disabled="isLoading"
                  >
                    🗑
                  </button>
                </div>
              </div>
              <div class="post-content">
                <p class="post-text">{{ post.content }}</p>
              </div>
              <div class="post-meta">
                <div class="post-date">
                  📅 {{ new Date(post.attributes.creation_date).toLocaleDateString() }}
                </div>
                <div
                  v-if="post.attributes.modified !== post.attributes.creation_date"
                  class="post-modified"
                >
                  ✏ {{ new Date(post.attributes.modified).toLocaleDateString() }}
                </div>
              </div>
            </div>
          </div>
          <p v-else class="no-posts">Aucun post disponible</p>

          <!-- Pagination -->
          <div v-if="totalPages > 1" class="pagination">
            <button
              @click="changePage(currentPage - 1)"
              :disabled="currentPage === 1 || isLoading"
              class="pagination-button"
            >
              Précédent
            </button>
            <span>Page {{ currentPage }} sur {{ totalPages }}</span>
            <button
              @click="changePage(currentPage + 1)"
              :disabled="currentPage === totalPages || isLoading"
              class="pagination-button"
            >
              Suivant
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<style scoped>
/* Global Styles */
body {
  font-family: Arial, sans-serif;
  background-color: #f4f7fa;
  margin: 0;
  padding: 0;
  color: #333;
}

h1,
h2 {
  font-size: 1.5rem;
  color: #333;
  margin-bottom: 1rem;
}

/* Posts Manager Container */
.posts-manager {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
}

/* Error Alert */
.error-alert {
  background-color: #f8d7da;
  color: #721c24;
  padding: 1rem;
  border: 1px solid #f5c6cb;
  margin-bottom: 1rem;
  position: relative;
  width: 100%;
  max-width: 600px;
  border-radius: 4px;
}

.error-alert .close-button {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  background: transparent;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
}

/* Loading Indicator */
.loading-indicator {
  font-size: 1.2rem;
  color: #007bff;
  margin-bottom: 1rem;
}

/* Form Section */
.form-section {
  background-color: #ffffff;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 600px;
}

.form-section .section-title {
  margin-bottom: 1rem;
  font-size: 1.8rem;
  text-align: center;
}

.form-group {
  margin-bottom: 1rem;
}

.form-input,
.form-textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
  font-size: 1rem;
}

.form-textarea {
  min-height: 150px;
  resize: vertical;
}

.character-count {
  font-size: 0.875rem;
  color: #666;
  text-align: right;
}

.button-group {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
}

.submit-button,
.cancel-button {
  padding: 0.75rem 1.25rem;
  font-size: 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.submit-button {
  background-color: #28a745;
  color: white;
}

.submit-button:disabled {
  background-color: #ddd;
}

.cancel-button {
  background-color: #dc3545;
  color: white;
}

.cancel-button:disabled {
  background-color: #ddd;
}

/* Posts Section */
.posts-section {
  width: 100%;
  max-width: 800px;
  margin-top: 2rem;
}

.posts-scrollable {
  overflow-y: auto;
  max-height: 500px;
}

.posts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.post-card {
  background-color: #fff;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  position: relative;
}

.post-card-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 1rem;
}

.post-title {
  font-size: 1.2rem;
  font-weight: bold;
  margin: 0;
}

.post-actions {
  display: flex;
  gap: 0.5rem;
}

.icon-button {
  background: transparent;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
}

.edit-button {
  color: #007bff;
}

.delete-button {
  color: #dc3545;
}

.post-content {
  font-size: 1rem;
  color: #555;
  margin-bottom: 1rem;
}

.post-meta {
  font-size: 0.875rem;
  color: #888;
  display: flex;
  justify-content: space-between;
}

.post-date {
  font-style: italic;
}

.post-modified {
  font-style: italic;
  color: #007bff;
}

/* No Posts Message */
.no-posts {
  text-align: center;
  color: #888;
  font-size: 1rem;
}

/* Pagination */
.pagination {
  display: flex;
  justify-content: center;
  margin-top: 2rem;
}

.pagination-button {
  padding: 0.5rem 1rem;
  font-size: 1rem;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.pagination-button:disabled {
  background-color: #ddd;
}
</style>
