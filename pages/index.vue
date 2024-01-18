<script setup>
	import { XMarkIcon } from "@heroicons/vue/24/outline";

	const config = useRuntimeConfig();

	import * as Realm from "realm-web";

	const app = new Realm.App({ id: "" });

	const {
		BSON: { ObjectID },
	} = Realm;

	const credentials = Realm.Credentials.anonymous();
	const user = await app.logIn(credentials);

	// console.log('hello', user.id === app.currentUser.id);

	const mongo = app.currentUser.mongoClient("mongodb-atlas");
	const collection = mongo.db("todo").collection("todos");

	const todos = ref([]);
	const enteredTodo = ref("");
	const beUpdated = ref(false);
	const beUpdatedTodoId = ref(null);

	// watch(enteredTodo, (newEnteredTodo) => {
	//   console.log('running');
	//   if (newEnteredTodo.length === 0) beUpdated.value = false;
	// });

	const resetTodo = () => (enteredTodo.value = "");

	const fetchTodos = async () => {
		const todosData = await collection.find();
		console.log(todosData);
		todos.value = todosData;
		console.log(todos);
	};

	onMounted(() => {
		fetchTodos();
	});

	const addTodoHandler = async () => {
		if (beUpdated.value) return;
		try {
			const newTodo = await collection.insertOne({ todo: enteredTodo.value });
			todos.value.push({ _id: newTodo.insertedId, todo: enteredTodo.value });
			resetTodo();
		} catch (err) {
			console.log(err);
			fetchTodos();
		}
	};

	const deleteTodoHandler = async (id) => {
		try {
			await collection.deleteOne({ _id: id });
			todos.value = todos.value.filter(
				(todo) => todo._id.toString() !== id.toString()
			);
		} catch (err) {
			console.log(err);
			fetchTodos();
		}
	};

	const updateTodo = (id) => {
		const todo = todos.value.find(
			(todo) => todo._id.toString() === id.toString()
		);
		enteredTodo.value = todo.todo;
		beUpdated.value = true;
		beUpdatedTodoId.value = id;

		console.log(todo);
	};

	const updateHandler = async (id) => {
		try {
			await collection.updateOne(
				{ _id: beUpdatedTodoId.value },
				{ todo: enteredTodo.value }
			);

			// setTodos((prevTodos) => {
			//     const filteredTodo = prevTodos.filter(
			//       (prevTodo) => prevTodo._id !== todo._id
			//     );

			//     return [...filteredTodo, todo];
			//   });

			const filteredTodo = todos.value.filter(
				(todo) => todo._id !== beUpdatedTodoId.value
			);

			todos.value = [
				...filteredTodo,
				{ todo: enteredTodo.value, _id: beUpdatedTodoId.value },
			];

			console.log(beUpdatedTodoId.value);
			resetTodo();
			beUpdated.value = false;
		} catch (err) {
			fetchTodos();
			console.log(err);
		}
	};

	const updateResetHandler = () => {
		beUpdated.value = false;
		beUpdatedTodoId.value = null;
		enteredTodo.value = "";
	};
</script>

<template>
	<main class="w-80 mx-auto">
		<div class="mt-20">
			<h2 class="mb-2 text-lg text-gray-500">Todo Items</h2>
			<form @submit.prevent="addTodoHandler" class="flex gap-2 w-full">
				<div class="w-full relative">
					<input
						type="text"
						placeholder="Type a todo..."
						v-model="enteredTodo"
						required
						class="px-2 py-2.5 rounded text-sm w-full focus:outline-none" />

					<XMarkIcon
						v-if="beUpdated"
						@click="updateResetHandler()"
						class="w-5 h-5 text-gray-500 cursor-pointer absolute right-2 top-3" />
				</div>

				<button
					v-if="beUpdated"
					@click="updateHandler"
					type="button"
					className="px-2 py-2 bg-green-600 rounded text-white hover:bg-green-700">
					Update
				</button>

				<button
					v-else
					type="submit"
					class="px-2 py-2 bg-orange-500 rounded text-white hover:bg-orange-600">
					Add
				</button>
			</form>
			<ul class="mt-8 flex flex-col-reverse gap-2">
				<li
					v-for="todo in todos"
					:key="todo._id"
					class="bg-white shadow px-2 py-2 rounded flex justify-between items-center">
					<p @click="updateTodo(todo._id)">{{ todo.todo }}</p>
					<button
						@click="deleteTodoHandler(todo._id)"
						type="button"
						class="text-xs">
						❌
					</button>
				</li>
			</ul>
		</div>
	</main>
</template>
