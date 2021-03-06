<template lang='pug'>
div#pet-profile
	section#pet-header
		section#pet-profile-picture-container
			img#pet-profile-picture(
				:src='picture'
			)
			b-field.file.is-primary(
				name='picture'
				native
				:class="{'has-name': !!file}"
			)
				b-upload.file-label( v-model='file' )
					span.file-cta
						b-icon.file-icon( icon='upload' )
						span.file-label(
							title='Upload Picture'
						) Click to upload
		section#pet-info
			h1.is-size-2 {{ pet.name }}
			h3.is-size-4 {{ pet.breed.name }}
			p.is-size-5 {{ pet.femaleOrMale }}
			hr
			section#owner-info
				h4.is-size-5
					| {{ owner.name }}
					br
					| {{ owner.phone }}
					br
					| {{ owner.address.street }}
					| {{ owner.address.number }}
					br
					| P.C. {{ owner.address.postalCode }}
	VaccinationRecord( :petId='pet._id' :records='pet.vaccinationRecord' )
	MedicalRecord
</template>

<script lang='js'>
import VaccinationRecord from './VaccinationRecord.vue'
import MedicalRecord from './MedicalRecord.vue'
import axios from 'axios'
import moment from 'moment'
import { EventBus } from '../eventBus.js'

export const OK = 200
export const CREATED = 201

export default {
	name: 'PetProfile',
	components: { VaccinationRecord, MedicalRecord },
	props: {
		id: String
	},
	data(){
		return{
			OK,
			CREATED,
			picture: process.env.VUE_APP_PLACEHOLDER,
			file: null,
			pet: {
				name: '',
				breed: {
					name: ''
				}
			},
			owner: {
				name: '',
				phone: '',
				address: {
					street: '',
					number: '',
					postalCode: ''
				}
			}
		}
	},
	computed: {
		user(){
			return this.$store.state.user
		}
	},
	watch: {
		/* We need a FormData object to send files over requests
		 * so we append our image to that object to send it. */
		file: function(){
			const formData = new FormData()
			formData.append('picture', this.file, this.file.name)
			formData.append('id', this.id)
			this.sendPicture(formData)
		}
	},
	methods: {
		init(){
			this.getPet()
		},
		updateProfilePicture(){
			this.picture = process.env.VUE_APP_TYRAWEB_PET_PROFILE_PICTURE + this.id + '.png'
		},
		getPet(){
			axios.get(process.env.VUE_APP_TYRAWEB_PET, {
				params: {
					id: this.id
				},
				headers: {
					Authorization: 'Bearer ' + this.user.token
				}
			}).then(response => {
				this.pet = response.data
				this.formatDates()
				this.isThereAProfilePicture()
				this.getOwner()
			}).catch(error => {
				console.error(error)
			})
		},
		getOwner(){
			axios.get(process.env.VUE_APP_TYRAWEB_CLIENT, {
				params: {
					id: this.pet.owner
				},
				headers: {
					Authorization: 'Bearer ' + this.user.token
				}
			}).then(response => {
				this.owner = response.data
			}).catch(error => {
				console.error(error)
			})
		},
		sendPicture(data){
			axios.post(process.env.VUE_APP_TYRAWEB_PET_ADD_PICTURE, data, {
				headers: {
					Authorization: 'Bearer ' + this.user.token,
					'Content-Type': 'multipart/form-data'
				}
			}).then(response => {
				if(response.status === this.CREATED)
					this.updateProfilePicture()
			}).catch(error => {
				console.error(error)
			})
		},
		isThereAProfilePicture(){
			axios.get(process.env.VUE_APP_TYRAWEB_PROFILE_PICTURE_EXISTS, {
				params: {
					id: this.id
				},
				headers: {
					Authorization: 'Bearer ' + this.user.token
				}
			}).then(response => {
				if(response.status === this.OK)
					this.updateProfilePicture()
			}).catch(error => {
				console.error(error)
			})
		},
		async formatDates(){
			await this.pet.vaccinationRecord.forEach(record => {
				record.applicationDate = moment(record.applicationDate).format('YYYY-MM-DD')
				record.nextApplicationDate = moment(record.nextApplicationDate).format('YYYY-MM-DD')
			})
		}
	},
	created(){
		this.init()

		/* Event Listeners */

		/* When a record is stored in the data base update the view. */
		EventBus.$on('update-vaccination-records', () => {
			this.getPet()
		})
	}
}
</script>

<style scoped>
@import '../assets/css/pet-profile.css'
</style>
