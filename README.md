 7: action_scale_recipe(); break;
            case 8: action_pantry_menu(); break;
            case 9: action_shopping_list(); break;
            case 0: save(); std::cout << "Goodbye!\n"; return;
            default: std::cout << "Invalid option.\n"; break;
        }
    }
}

void CookBookApp::action_add_recipe(){
    Recipe r; r.name = prompt_str("Recipe name");
    r.servings = prompt_int("Servings", 2);
    std::string tagLine = prompt_str("Tags (comma-separated)");
    r.tags = util::split(tagLine, ',', false);

    std::cout << "Add ingredients (blank name to stop)\n";
    while(true){
        std::string in = prompt_str("Ingredient name", "");
        if(in.empty()) break;
        Ingredient ing; ing.name = in;
        ing.quantity = prompt_double("Quantity", 1.0);
        ing.unit = prompt_str("Unit (g, ml, tbsp, pcs, etc.)", "pcs");
        r.ingredients.push_back(ing);
    }

    std::cout << "Enter steps (blank line to finish)\n";
    while(true){
        std::string step = prompt_str("Step", "");
        if(step.empty()) break;
        r.steps.push_back(step);
    }

    data_.recipes.push_back(r);
    save();
    std::cout << "[+] Recipe added.\n";
}

void CookBookApp::action_list_recipes(){
    if(data_.recipes.empty()){ std::cout << "No recipes yet.\n"; return; }
    std::cout << "\nRecipes (" << data_.recipes.size() << "):\n";
    for(const auto& r : data_.recipes){
        std::cout << " - " << r.name << " [serves " << r.servings << "] ";
        if(!r.tags.empty()) std::cout << "{" << util::join(r.tags, ", ") << "}";
        std::cout << "\n";
    }
}

void CookBookApp::action_view_recipe(){
    auto idx = pick_recipe_index("View which?"); if(!idx) return;
    const auto& r = data_.recipes[*idx];
    std::cout << "\n== " << r.name << " ==\nServings: " << r.servings << "\nTags: " << (r.tags.empty()?"-":util::join(r.tags, ", ")) << "\n\nIngredients:\n";
    for(const auto& ing : r.ingredients){
        std::cout << " * " << ing.name << ": " << ing.quantity << " " << ing.unit << "\n";
    }
    std::cout << "\nSteps:\n";
    for(size_t i=0;i<r.steps.size();++i){
        std::cout << i+1 << ") " << r.steps[i] << "\n";
    }
}

void CookBookApp::action_edit_recipe(){
    auto idx = pick_recipe_index("Edit which?"); if(!idx) return;
    auto& r = data_.recipes[*idx];
    r.name = prompt_str("Name", r.name);
    r.servings = prompt_int("Servings", r.servings);
    std::string tagLine = prompt_str("Tags (comma, empty to keep)", util::join(r.tags, ","));
    r.tags = util::split(tagLine, ',', false);

    std::cout << "Edit ingredients? (y/N): "; std::string yn; std::getline(std::cin, yn);
    if(!yn.empty() && (yn[0]=='y' || yn[0]=='Y')){
        r.ingredients.clear();
        std::cout << "Re-enter ingredients (blank to stop)\n";
        while(true){
            std::string in = prompt_str("Ingredient name", "");
            if(in.empty()) break;
            Ingredient ing; ing.name = in;
            ing.quantity = prompt_double("Quantity", 1.0);
            ing.unit = prompt_str("Unit", "pcs");
            r.ingredients.push_back(ing);
        }
    }

    std::cout << "Edit steps? (y/N): "; std::getline(std::cin, yn);
    if(!yn.empty() && (yn[0]=='y' || yn[0]=='Y')){
        r.steps.clear();
        std::cout << "Re-enter steps (blank to stop)\n";
        while(true){
            std::string step = prompt_str("Step", "");
            if(step.empty()) break;
            r.steps.push_back(step);
        }
    }
    save();
    std::cout << "[+] Updated.\n";
}

void CookBookApp::action_remove_recipe(){
    auto idx = pick_recipe_index("Remove which?"); if(!idx) return;
    std::cout << "Are you sure? (y/N): "; std::string yn; std::getline(std::cin, yn);
    if(!yn.empty() && (yn[0]=='y' || yn[0]=='Y')){
        data_.recipes.erase(data_.recipes.begin() + *idx);
        save();
        std::cout << "[-] Removed.\n";
    } else {
        std::cout << "Cancelled.\n";
    }
}

void CookBookApp::action_search(){
    std::string q = prompt_str("Search term (name/ingredient/tag)");
    auto ql = util::tolower_copy(q);
    int count=0;
    for(const auto& r : data_.recipes){
        bool hit=false;
        if(util::tolower_copy(r.name).find(ql) != std::string::npos) hit = true;
        for(const auto& t : r.tags){ if(util::tolower_copy(t).find(ql)!=std::string::npos) hit=true; }
        for(const auto& ing : r.ingredients){ if(util::tolower_copy(ing.name).find(ql)!=std::string::npos) hit=true; }
        if(hit){
            std::cout << " - " << r.name << " [serves " << r.servings << "]\n"; ++count;
        }
    }
    if(!count) std::cout << "No matches.\n";
}

void CookBookApp::action_scale_recipe(){
    auto idx = pick_recipe_index("Scale which?"); if(!idx) return;
    const auto& r = data_.recipes[*idx];
    int target = prompt_int("Target servings", r.servings);
    if(target <= 0){ std::cout << "Invalid target.\n"; return; }
    double factor = (double)target / (double)r.servings;
    std::cout << "\nScaled ingredients for " << r.name << " (serves " << target << "):\n";
    for(const auto& ing : r.ingredients){
        std::cout << " * " << ing.name << ": " << ing.quantity * factor << " " << ing.unit << "\n";
    }
}

void CookBookApp::action_pantry_menu(){
    while(true){
        std::cout << "\n-- Pantry --\n";
        std::cout << "1) Add/Update item\n2) Remove item\n3) List pantry\n0) Back\n";
        int c = prompt_int("Choose", 3);
        if(c==1) action_pantry_add();
        else if(c==2) action_pantry_remove();
        else if(c==3) action_pantry_list();
        else if(c==0) return;
        else std::cout << "Invalid.\n";
    }
}

void CookBookApp::action_pantry_add(){
    std::string name = prompt_str("Item name");
    double qty = prompt_double("Quantity", 1.0);
    std::string unit = prompt_str("Unit", "pcs");
    std::string key = util::tolower_copy(name);
    data_.pantry[key] = PantryItem{name, qty, unit};
    save();
    std::cout << "[+] Pantry updated.\n";
}

void CookBookApp::action_pantry_remove(){
    std::string name = prompt_str("Item to remove");
    std::string key = util::tolower_copy(name);
    auto it = data_.pantry.find(key);
    if(it == data_.pantry.end()){ std::cout << "Not found.\n"; return; }
    data_.pantry.erase(it); save();
    std::cout << "[-] Removed from pantry.\n";
}

void CookBookApp::action_pantry_list(){
    if(data_.pantry.empty()){ std::cout << "Pantry empty.\n"; return; }
    std::cout << "Pantry items (" << data_.pantry.size() << "):\n";
    for(const auto& kv : data_.pantry){
        const auto& p = kv.second;
        std::cout << " - " << p.name << ": " << p.quantity << " " << p.unit << "\n";
    }
}

void CookBookApp::action_shopping_list(){
    std::vector<int> chosen;
    std::cout << "Select recipes to cook (enter numbers separated by spaces, or blank for all):\n";
    for(size_t i=0;i<data_.recipes.size();++i){ std::cout << i+1 << ") " << data_.recipes[i].name << "\n"; }
    std::cout << "> ";
    std::string line; std::getline(std::cin, line); line = util::trim(line);
    if(line.empty()){
        for(size_t i=0;i<data_.recipes.size();++i) chosen.push_back((int)i);
    } else {
        auto parts = util::split(line, ' ', false);
        for(const auto& p : parts){
            try{ int idx = std::stoi(p)-1; if(idx>=0 && idx<(int)data_.recipes.size()) chosen.push_back(idx); }catch(...){ }
        }
    }
    if(chosen.empty()){ std::cout << "No recipes chosen.\n"; return; }

    // Aggregate needs
    struct Need { double qty{0}; std::string unit; };
    std::unordered_map<std::string, Need> needs; // key: lower ingredient name

    for(int idx : chosen){
        const auto& r = data_.recipes[idx];
        for(const auto& ing : r.ingredients){
            auto key = util::tolower_copy(ing.name);
            auto& n = needs[key];
            if(n.unit.empty()) n.unit = ing.unit;
            // NOTE: No unit conversion across differing units (kept simple)
            n.qty += ing.quantity;
        }
    }

    // Subtract pantry
    struct Missing { double qty{0}; std::string unit; std::string name; double have{0}; };
    std::vector<Missing> missing;
    for(auto& kv : needs){
        const std::string& key = kv.first; auto& need = kv.second;
        double have = 0.0; std::string unit = need.unit;
        auto pit = data_.pantry.find(key);
        if(pit != data_.pantry.end() && util::iequals(pit->second.unit, unit)){
            have = pit->second.quantity;
        }
        double shortfall = need.qty - have;
        if(shortfall > 0.0001){
            Missing m; m.name = data_.pantry.count(key)? data_.pantry[key].name : key; // preserve original case if present
            m.qty = shortfall; m.unit = unit; m.have = have;
            missing.push_back(m);
        }
    }

    // Output
    std::cout << "\n=== Shopping List ===\n";
    if(missing.empty()){
        std::cout << "You have everything you need.\n";
    } else {
        for(const auto& m : missing){
            std::cout << " - " << m.name << ": need " << m.qty << " " << m.unit;
            if(m.have > 0) std::cout << " (have " << m.have << ")";
            std::cout << "\n";
        }
    }
}

# ---------- src/main.cpp ----------
#include "cookbook.hpp"
#include <filesystem>
#include <iostream>

int main(){
    std::string dbPath = "data/cookbook.db";
    try{ std::filesystem::create_directories("data"); }catch(...){ }
    CookBookApp app(dbPath);
    std::cout << "Data file: " << dbPath << "\n";
    app.run();
    return 0;
}

# ---------- data/cookbook.db (starter sample) ----------
#RECIPES
R|Simple Omelette|2|breakfast,quick|Eggs@3@pcs,Milk@30@ml,Salt@0.25@tsp,Butter@10@g|Beat eggs with milk and salt\nHeat pan and melt butter\nCook eggs, fold, and serve
R|Tomato Pasta|2|dinner,veg|Pasta@200@g,Tomato sauce@150@ml,Olive oil@1@tbsp,Garlic@2@pcs,Salt@0.5@tsp|Boil pasta until al dente\nSaute garlic in oil\nAdd sauce, toss pasta, season to taste
#PANTRY
P|Salt|1|tsp
P|Olive oil|2|tbsp
