# mapup-assignment-package main

import (
	"encoding/json"
	"log"
	"net/http"
	"sort"
	"sync"
	"time"
)

// SortRequest structure for JSON payload
type SortRequest struct {
	ToSort [][]int json:"to_sort"
}

// SortResponse structure
type SortResponse struct {
	SortedArrays [][]int json:"sorted_arrays"
	TimeNs       int64   json:"time_ns"
}

func main() {
	http.HandleFunc("/process-single", processSingle)
	http.HandleFunc("/process-concurrent", processConcurrent)

	// Listen and serve on port 8000
	log.Fatal(http.ListenAndServe(":8000", nil))
}

func processSingle(w http.ResponseWriter, r *http.Request) {
	var req SortRequest
	err := json.NewDecoder(r.Body).Decode(&req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	startTime := time.Now()
	sortedArrays := make([][]int, len(req.ToSort))

	for i, arr := range req.ToSort {
		sort.Ints(arr)
		sortedArrays[i] = arr
	}

	timeTaken := time.Since(startTime).Nanoseconds()

	resp := SortResponse{
		SortedArrays: sortedArrays,
		TimeNs:       timeTaken,
	}

	sendResponse(w, resp)
}

func processConcurrent(w http.ResponseWriter, r *http.Request) {
	var req SortRequest
	err := json.NewDecoder(r.Body).Decode(&req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	startTime := time.Now()
	sortedArrays := make([][]int, len(req.ToSort))

	var wg sync.WaitGroup
	for i, arr := range req.ToSort {
		wg.Add(1)
		go func(i int, arr []int) {
			defer wg.Done()
			sort.Ints(arr)
			sortedArrays[i] = arr
		}(i, arr)
	}

	wg.Wait()

	timeTaken := time.Since(startTime).Nanoseconds()

	resp := SortResponse{
		SortedArrays: sortedArrays,
		TimeNs:       timeTaken,
	}

	sendResponse(w, resp)
}

func sendResponse(w http.ResponseWriter, resp SortResponse) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)

	err := json.NewEncoder(w).Encode(resp)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
}
