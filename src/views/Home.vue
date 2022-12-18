<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useSignOut, useUserId } from '@nhost/vue'
import { useMutation, useQuery } from '@vue/apollo-composable'
import { gql } from '@apollo/client/core'

const router = useRouter()
const { signOut } = useSignOut()
const userId = useUserId()

const newNote = ref({
	title: '',
	content: ''
})

const logout = () => {
	signOut()

	router.push('/login')
}

const {
	loading: notesLoading, 
	result: notesData, 
	refetch: notesRefetch 
} = useQuery(gql`
	query ($userId: String!) {
		notes (
			order_by: { created: desc }, 
			where: { user_id: {_eq: $userId} }
		) {
			id
			title,
			content,
			created
		}
	}
`, { userId: userId.value })

const { mutate: deleteNote, onDone: deleteDone } = useMutation(gql`
	mutation ($id: Int!) {
		delete_notes(where: {id: {_eq: $id}}) {affected_rows}
	}
`)

deleteDone(() => {
	notesRefetch()
	console.log(notesData);
})

const { 
	mutate: createNote, 
	onDone: createDone 
} = useMutation(gql`
	mutation ($userId: String!, $title: String!, $content: String!) {
		insert_notes_one(object: {user_id: $userId, title: $title, content: $content}) {
			id
		}
	}
`)

const handleCreate = () => {
	if (!newNote.value.title || !newNote.value.content) {
		return alert("Please fill in all fields")
	}

	createNote({
		userId: userId.value,
		title: newNote.value.title,
		content: newNote.value.content
	})
}

createDone(() => {
	notesRefetch()
	newNote.value = {
		title: '',
		content: ''
	}
})

const convertToHtml = (content) => {
	return content.replace(/\n/g, '<br />')
} 

const convertToDate = (date) => {
	return new Date(date).toLocaleString()
}

</script>

<template>
	<main>
		<div class="flex items-center justify-between mb-8">
			<h1 class="text-4xl font-bold">My Notes App</h1>
			<button @click="logout" class="text-red-500 hover:underline cursor-pointer">Logout</button>
		</div>

		<form @submit.prevent="handleCreate">
			<label class="block mb-4">
				<span class="block text-sm uppercase mb-2">Title</span>
				<input type="text" v-model="newNote.title" class="block w-full text-slate-800 px-4 py-2" />
			</label>

			<label class="block mb-4">
				<span class="block text-sm uppercase mb-2">Content</span>
				<textarea v-model="newNote.content" class="block w-full text-slate-800 px-4 py-2"></textarea>
			</label>

			<input 
				type="submit"
				value="Create note"
				class="text-green-500 hover:underline mb-4 cursor-pointer" />
		</form>

		<div v-if="!notesLoading">
			<div v-for="note in notesData.notes" class="relative bg-white text-slate-700 rounded-lg p-6 mb-6" :key="note">
				<button class="text-red-500 font-bold absolute top-6 right-6" @click="() => deleteNote({id: note.id})">delete</button>
				<h3 class="font-bold text-2xl mb-4">{{ note.title }}</h3>
				<p 
					class="text-lg text-gray-500 mb-4" 
					v-html="convertToHtml(note.content)"></p>
				<div class="text-sm text-gray-500 italic">
					{{ convertToDate(note.created) }}
					</div>
			</div>
		</div>
		<div v-else>Loading notes...</div>
	</main>
</template>