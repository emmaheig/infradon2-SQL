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
/* Variables CSS */
:root {
  --primary-color: #4caf50; /* Vert pour les boutons */
  --secondary-color: #363636; /* Texte secondaire sombre */
  --background-color: #ffffff; /* Couleur de fond */
  --text-color: #111; /* Texte principal très foncé */
  --error-color: #f44336; /* Rouge pour les erreurs */
  --border-radius: 8px; /* Coins arrondis */
  --transition-speed: 0.3s; /* Vitesse des transitions */
  --font-family: 'Arial', sans-serif; /* Police principale */
  --max-width: 900px; /* Largeur maximale de la colonne */
}

/* Global */
body {
  font-family: var(--font-family);
  background-color: var(--background-color);
  color: #000000;
  margin: 0;
  padding: 0;
}

h1, h2, h3 {
  margin: 0 0 1rem;
  color:  #000000;
}

/* Container principal */
.posts-manager {
  max-width: var(--max-width);
  margin: 2rem auto;
  padding: 1.5rem;
  background: #fff;
  border-radius: var(--border-radius);
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* Section de formulaire */
.form-section {
  margin-bottom: 2rem;
}

.form-group {
  margin-bottom: 1rem;
  display: flex;
  flex-direction: column;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 1rem; /* Augmentation pour un design plus confortable */
  border: 1px solid #ccc;
  border-radius: var(--border-radius);
  font-size: 1rem;
}

.form-group input:focus,
.form-group textarea:focus {
  outline: none;
  border-color: var(--primary-color);
}

.character-count {
  font-size: 0.875rem;
  color: #302f2f;
  margin-top: 0.25rem;
}

.button-group {
  display: flex;
  gap: 0.75rem;
}

button {
  padding: 1rem 1.5rem; /* Boutons plus grands */
  font-size: 1.1rem; /* Texte légèrement agrandi */
  color: #fff;
  background-color: #4caf50;
  border: none;
  border-radius: 30px;
  cursor: pointer;
  transition: transform var(--transition-speed);
}

button:disabled {
  background-color: hwb(146 0% 0%);
  cursor: not-allowed;
}

.cancel-button {
  background-color: var(--error-color);
}

.cancel-button:disabled {
  background-color: #e57373;
}

/* Liste des posts */
.posts-section {
  margin-top: 2.5rem;
}

.posts-scrollable {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.post-card {
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: var(--border-radius);
  background: #fdfdfd;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.post-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.post-title {
  font-size: 1.25rem;
  font-weight: bold;
  color:  #302f2f;
}

.post-actions {
  display: flex;
  gap: 0.5rem;
}

.post-actions button {
  padding: 0.5rem;
  font-size: 1rem;
  border-radius: var(--border-radius);
}

/* Texte du post */
.post-content {
  margin: 0.5rem 0;
  color: var(--secondary-color);
}

.post-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.9rem;
  color: #0e0d0d;
}

/* Pagination */
.pagination {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 2rem;
}

.pagination-button {
  padding: 0.75rem 1.25rem;
  font-size: 1rem;
  background-color: var(--primary-color);
  color: #fff;
  border: none;
  border-radius: var(--border-radius);
  cursor: pointer;
}

.pagination-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
</style>

