<script lang="ts">
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'
import Post from './Interface'

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
    this.createData({
      _id: '1',
      doc: {
        post_name: 'Post 1',
        post_content: 'Contenu du post 1',
        attributes: {
          creation_date: '2021-09-01',
          modified: 'not yet',
        },
      },
    })
    this.updateData({
      _id: '1',
      doc: {
        post_name: 'Post 1',
        post_content: 'Contenu du post 1',
        attributes: {
          creation_date: '2021-09-01',
          modified: 'yes',
        },
      },
    })
  },

  methods: {
    async updateData(document: Post) {
      const db = this.storage

      if (!db) {
        console.error("Le stockage n'est pas défini.")
        return
      }

      try {
        const existingDoc = await db.get(document._id)
        document._rev = existingDoc._rev
        await db.put(document)
        console.log('Mise à jour réussie')
        this.fetchData()
      } catch (error) {
        console.error('Erreur lors de la mise à jour', error)
      }
    },

    async deleteData(document: Post) {
      const db = this.storage

      if (!db) {
        console.error("Le stockage n'est pas défini.")
        return
      }

      try {
        const existingDoc = await db.get(document._id)
        await db.remove(existingDoc)
        console.log('Suppression réussie')
        this.fetchData()
      } catch (error) {
        console.error('Erreur lors de la suppression', error)
      }
    },

    async fetchData() {
      if (!this.storage) {
        console.error("Le stockage n'est pas défini.")
        return
      }

      try {
        const result = await this.storage.allDocs({ include_docs: true })
        this.postsData = result.rows.map(row => row.doc as Post)
        console.log('Documents récupérés avec succès')
      } catch (error) {
        console.error('Erreur lors de la récupération des documents', error)
      }
    },

    async createData(document: Post) {
      if (!this.storage) {
        console.error("Le stockage n'est pas défini.")
        return
      }

      try {
        await this.storage.put(document)
        console.log('Document ajouté avec succès')
        this.fetchData()
      } catch (error) {
        console.error("Erreur lors de l'ajout du document", error)
      }
    },

    initDatabase() {
      this.storage = new PouchDB('http://admin:ecs@localhost:5984/database')
      if (this.storage) {
        console.log('Connexion établie avec la base de données distante.')
      } else {
        console.warn('La connexion à la base de données a échoué.')
      }
    },
  },
}
</script>

<template>
  <div>
    <h1>Nombre de posts : {{ postsData.length }}</h1>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <div class="ucfirst">
          {{ post.doc.post_name }}
          <em
            style="font-size: x-small"
            v-if="post.doc.attributes?.creation_date"
          >
            - {{ post.doc.attributes.creation_date }}
          </em>
        </div>
        <button @click="deleteData(post)">Supprimer</button>
      </li>
    </ul>
    <button
      @click="
        createData({
          _id: new Date().toISOString(),
          doc: {
            post_name: 'Nouveau Post',
            post_content: 'Contenu par défaut',
            attributes: {
              creation_date: new Date().toISOString(),
              modified: 'not yet',
            },
          },
        })
      "
    >
      Ajouter un post
    </button>
  </div>
</template>
