# recipe-tracker-app
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

// --- Domain Models ---
data class Ingredient(val name: String, val quantity: String)
data class Recipe(val id: String, val title: String, val instructions: String, val ingredients: List<Ingredient>, val isFavorite: Boolean = false)

// --- Repository Abstraction (Decouples data sources from UI logic) ---
interface RecipeRepository {
    suspend fun getAllRecipes(): List<Recipe>
    suspend fun saveRecipe(recipe: Recipe)
    suspend fun deleteRecipe(id: String)
}

// --- In-Memory implementation of the Repository ---
class InMemoryRecipeRepository : RecipeRepository {
    private val database = mutableMapOf<String, Recipe>()

    override suspend fun getAllRecipes(): List<Recipe> = database.values.toList()

    override suspend fun saveRecipe(recipe: Recipe) {
        database[recipe.id] = recipe
    }

    override suspend fun deleteRecipe(id: String) {
        database.remove(id)
    }
}

// --- State Wrapper for UI/API consumption ---
data class RecipeUiState(
    val recipes: List<Recipe> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

// --- Business Logic / Presenter Layer ---
class RecipeViewModel(private val repository: RecipeRepository) {
    
    private val _uiState = MutableStateFlow(RecipeUiState())
    val uiState: StateFlow<RecipeUiState> = _uiState.asStateFlow()

    suspend fun loadRecipes() {
        _uiState.update { it.copy(isLoading = true) }
        try {
            val list = repository.getAllRecipes()
            _uiState.update { it.copy(recipes = list, isLoading = false) }
        } catch (e: Exception) {
            _uiState.update { it.copy(errorMessage = e.localizedMessage, isLoading = false) }
        }
    }

    suspend fun addNewRecipe(title: String, instructions: String, ingredients: List<Ingredient>) {
        val uniqueId = java.util.UUID.randomUUID().toString()
        val newRecipe = Recipe(uniqueId, title, instructions, ingredients)
        
        repository.saveRecipe(newRecipe)
        loadRecipes() // Refresh state
    }
}
