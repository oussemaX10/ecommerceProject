#!/bin/bash

# Pre-commit hook to enforce checkstyle rules
# This script blocks commits that have unused imports, System.out.println, or other violations

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}🔍 Running pre-commit checks for ERP project...${NC}"

# Check if we're in a git repository
if ! git rev-parse --git-dir > /dev/null 2>&1; then
    echo -e "${RED}❌ Not in a git repository${NC}"
    exit 1
fi

# Get the list of staged Java files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(java)$')

if [ -z "$STAGED_FILES" ]; then
    echo -e "${GREEN}✅ No Java files to check${NC}"
    exit 0
fi

echo -e "${YELLOW}📝 Checking staged Java files:${NC}"
echo "$STAGED_FILES"

# Function to check for unused imports in staged files
check_unused_imports() {
    local has_violations=false
    
    echo -e "\n${YELLOW}🔍 Checking for unused imports...${NC}"
    
    for file in $STAGED_FILES; do
        if [ -f "$file" ]; then
            # Quick check for potential unused imports
            # This is a simplified check - the full checkstyle will catch more cases
            imports=$(grep -n "^import " "$file" | grep -v "import static")
            
            if [ -n "$imports" ]; then
                while IFS= read -r import_line; do
                    # Extract the import statement
                    import_class=$(echo "$import_line" | sed 's/.*import \([^;]*\);.*/\1/' | sed 's/.*\.//')
                    
                    # Check if the imported class is used in the file
                    if ! grep -q "$import_class" "$file" --exclude-dir=target; then
                        # Additional check to avoid false positives
                        simple_name=$(echo "$import_class" | sed 's/.*\.//')
                        if ! grep -q "$simple_name" "$file"; then
                            echo -e "${RED}❌ Potentially unused import in $file:${NC}"
                            echo -e "   $import_line"
                            has_violations=true
                        fi
                    fi
                done <<< "$imports"
            fi
        fi
    done
    
    return $has_violations
}

# Function to check for System.out.println statements
check_system_out() {
    local has_violations=false
    
    echo -e "\n${YELLOW}🔍 Checking for System.out/System.err statements...${NC}"
    
    for file in $STAGED_FILES; do
        if [ -f "$file" ]; then
            # Check for System.out.print*, System.err.print*, printStackTrace
            violations=$(grep -n -E "(System\.(out|err)\.print|\.printStackTrace\(\))" "$file" | grep -v "//")
            
            if [ -n "$violations" ]; then
                echo -e "${RED}❌ System.out/System.err violations found in $file:${NC}"
                echo "$violations" | while IFS= read -r line; do
                    echo -e "   ${RED}$line${NC}"
                done
                has_violations=true
            fi
        fi
    done
    
    return $has_violations
}

# Function to check for hardcoded secrets
check_hardcoded_secrets() {
    local has_violations=false
    
    echo -e "\n${YELLOW}🔍 Checking for hardcoded secrets...${NC}"
    
    for file in $STAGED_FILES; do
        if [ -f "$file" ]; then
            # Check for potential hardcoded secrets
            secrets=$(grep -n -E "(password|secret|key|token).*=.*[\"'][^$]" "$file" | grep -v "//")
            
            if [ -n "$secrets" ]; then
                echo -e "${RED}❌ Potential hardcoded secrets found in $file:${NC}"
                echo "$secrets" | while IFS= read -r line; do
                    echo -e "   ${RED}$line${NC}"
                done
                has_violations=true
            fi
        fi
    done
    
    return $has_violations
}

# Function to run full checkstyle validation
run_checkstyle() {
    echo -e "\n${YELLOW}🔍 Running full Checkstyle validation...${NC}"
    
    # Check if Maven is available
    if command -v mvn > /dev/null 2>&1; then
        echo -e "${YELLOW}📦 Using Maven to run Checkstyle...${NC}"
        
        # Run checkstyle with Maven
        mvn checkstyle:check -q
        checkstyle_exit_code=$?
        
        if [ $checkstyle_exit_code -ne 0 ]; then
            echo -e "${RED}❌ Checkstyle validation failed!${NC}"
            echo -e "${RED}📋 Check the output above for detailed violations${NC}"
            return 1
        fi
        
    # Check if Gradle is available
    elif command -v gradle > /dev/null 2>&1 || [ -f "./gradlew" ]; then
        echo -e "${YELLOW}📦 Using Gradle to run Checkstyle...${NC}"
        
        # Run checkstyle with Gradle
        if [ -f "./gradlew" ]; then
            ./gradlew checkstyleMain checkstyleTest -q
        else
            gradle checkstyleMain checkstyleTest -q
        fi
        
        checkstyle_exit_code=$?
        
        if [ $checkstyle_exit_code -ne 0 ]; then
            echo -e "${RED}❌ Checkstyle validation failed!${NC}"
            echo -e "${RED}📋 Check the output above for detailed violations${NC}"
            return 1
        fi
        
    else
        echo -e "${YELLOW}⚠️ Neither Maven nor Gradle found, skipping full checkstyle validation${NC}"
        echo -e "${YELLOW}💡 Install Maven or Gradle for complete validation${NC}"
        return 0
    fi
    
    echo -e "${GREEN}✅ Checkstyle validation passed!${NC}"
    return 0
}

# Function to run tests
run_tests() {
    echo -e "\n${YELLOW}🧪 Running tests...${NC}"
    
    # Run tests with Maven
    if command -v mvn > /dev/null 2>&1; then
        mvn test -q
        test_exit_code=$?
        
        if [ $test_exit_code -ne 0 ]; then
            echo -e "${RED}❌ Tests failed!${NC}"
            return 1
        fi
        
    # Run tests with Gradle
    elif command -v gradle > /dev/null 2>&1 || [ -f "./gradlew" ]; then
        if [ -f "./gradlew" ]; then
            ./gradlew test -q
        else
            gradle test -q
        fi
        
        test_exit_code=$?
        
        if [ $test_exit_code -ne 0 ]; then
            echo -e "${RED}❌ Tests failed!${NC}"
            return 1
        fi
    else
        echo -e "${YELLOW}⚠️ Neither Maven nor Gradle found, skipping tests${NC}"
        return 0
    fi
    
    echo -e "${GREEN}✅ All tests passed!${NC}"
    return 0
}

# Main execution
main() {
    local has_violations=false
    
    # Quick checks on staged files
    if ! check_unused_imports; then
        has_violations=true
    fi
    
    if ! check_system_out; then
        has_violations=true
    fi
    
    if ! check_hardcoded_secrets; then
        has_violations=true
    fi
    
    # If quick checks failed, don't proceed
    if [ "$has_violations" = true ]; then
        echo -e "\n${RED}❌ COMMIT BLOCKED: Fix the violations above before committing${NC}"
        echo -e "${YELLOW}💡 Tips:${NC}"
        echo -e "   • Remove unused imports using your IDE (Ctrl+Shift+O in IntelliJ/Eclipse)"
        echo -e "   • Replace System.out.println with proper logging (logger.info/debug/error)"
        echo -e "   • Move hardcoded secrets to environment variables or application.properties"
        echo -e "   • Use @Value annotations to inject configuration values"
        exit 1
    fi
    
    # Run full checkstyle validation
    if ! run_checkstyle; then
        echo -e "\n${RED}❌ COMMIT BLOCKED: Checkstyle validation failed${NC}"
        echo -e "${YELLOW}💡 Run 'mvn checkstyle:check' or 'gradle checkstyle' for detailed report${NC}"
        exit 1
    fi
    
    # Run tests (optional - comment out if you want faster commits)
    # if ! run_tests; then
    #     echo -e "\n${RED}❌ COMMIT BLOCKED: Tests failed${NC}"
    #     exit 1
    # fi
    
    echo -e "\n${GREEN}✅ All pre-commit checks passed! Proceeding with commit...${NC}"
    exit 0
}

# Run the main function
main
