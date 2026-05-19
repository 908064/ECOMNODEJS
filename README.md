"use client";

import React, { useState, useEffect } from "react";
import {
  Plus,
  Trash2,
  GripVertical,

  Copy,

  Link,
  ArrowLeft,
  Type,
  AlignLeft,
  Mail,
  Hash,
  CircleDot,
  CheckSquare,
  ChevronDown,
  Upload,
  File,
  ChevronDownCircle,
  FileDigit,
  Save,
  Menu,
  X,
  Clock,
  Trophy,
  Eye,
  Edit,
  MoreVertical,
  ChevronsUpDown,
} from "lucide-react";
import {
  DndContext,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
  DragEndEvent,
} from "@dnd-kit/core";
import {
  arrayMove,
  SortableContext,
  sortableKeyboardCoordinates,
  verticalListSortingStrategy,
} from "@dnd-kit/sortable";
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { restrictToParentElement } from "@dnd-kit/modifiers";
import { useAuth } from "@/contexts/AuthContext";
import { updateAptitudeTestTemplate, createAptitudeTestTemplate } from "@/config/apiService";
import { useRouter } from "next/navigation";
import { GraphQLClient } from "@/lib/utils";
import { config } from "@/config";
import { ProtectedRoute } from "@/components/ProtectedRoute";
import { toast } from "react-toastify";
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip";

// Types
type FieldType =
  | "text"
  | "longText"
  | "email"
  | "number"
  | "multipleChoice"
  | "checkboxes"
  | "dropdown"
  | "fileUpload";

type SectionType = "logic_reasoning" | "coding_assessment" | "quantitative_ability" | "custom";

interface Option {
  id: string;
  value: string;
}

interface FormSection {
  id: string;
  type: SectionType;
  title: string;
  description?: string;
  order: number;
  fields: FormField[];
}

interface FormField {
  id: string;
  type: FieldType;
  sectionId: string;
  label: string;
  required: boolean;
  options?: Option[];
  answer?: string;
  score?: number;
}

interface SavedForm {
  id: string;
  title: string;
  description?: string;
  timeLimitMinutes?: number;
  fields: FormField[];
  createdAt: Date;
}

interface FormResponse {
  [fieldId: string]: any;
}

type ViewMode = "builder" | "preview" | "fillForm";

interface TestFormBuilderProps {
  id?: string;
}

interface DatabaseField extends FormField {
  dbId?: string; // Database ID for existing fields
}

// Default sections configuration
const DEFAULT_SECTIONS: FormSection[] = [
  {
    id: "section_logic_reasoning",
    type: "logic_reasoning",
    title: "Logic Reasoning",
    description: "Test your logical thinking and reasoning abilities",
    order: 1,
    fields: [],
  },
  {
    id: "section_coding_assessment",
    type: "coding_assessment",
    title: "Coding Assessment",
    description: "Demonstrate your programming skills and problem-solving",
    order: 2,
    fields: [],
  },
  {
    id: "section_quantitative_ability",
    type: "quantitative_ability",
    title: "Quantitative Ability",
    description: "Assess mathematical and numerical skills",
    order: 3,
    fields: [],
  },
];

// Section type colors and icons
// const SECTION_CONFIG: Record<SectionType, { color: string; icon: string; bgColor: string }> = {
//   logic_reasoning: {
//     color: "bg-blue-100 text-blue-700 border-blue-300",
//     icon: "🧠",
//     bgColor: "bg-blue-50",
//   },
//   coding_assessment: {
//     color: "bg-green-100 text-green-700 border-green-300",
//     icon: "💻",
//     bgColor: "bg-green-50",
//   },
//   quantitative_ability: {
//     color: "bg-purple-100 text-purple-700 border-purple-300",
//     icon: "📊",
//     bgColor: "bg-purple-50",
//   },
//   custom: {
//     color: "bg-gray-100 text-gray-700 border-gray-300",
//     icon: "📝",
//     bgColor: "bg-gray-50",
//   },
// };

// Sortable Field Component
const SortableField = ({
  field,
  renderField,
}: {
  field: DatabaseField;
  renderField: (field: DatabaseField) => React.ReactNode;
}) => {
  const {
    attributes,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id: field.id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.5 : 1,
  };

  return (
    <div ref={setNodeRef} style={style} {...attributes}>
      {renderField(field)}
    </div>
  );
};

const TestFormBuilder = ({ id }: TestFormBuilderProps) => {
  const { tokens } = useAuth();

  return (
    <ProtectedRoute allowedRoles={["Recruiter"]}>
      <TestFormBuilderContent id={id} />
    </ProtectedRoute>
  );
};

const TestFormBuilderContent = ({ id }: TestFormBuilderProps) => {
  const { tokens } = useAuth();
  const [isLoading, setIsLoading] = useState(!id); // Show loading if no id provided
  const [viewMode, setViewMode] = useState<ViewMode>("builder");
  const [isPreviewMode, setIsPreviewMode] = useState(false);
  const [formTitle, setFormTitle] = useState("Aptitude Test Form");
  const [formDescription, setFormDescription] = useState("");
  const [timeLimitMinutes, setTimeLimitMinutes] = useState<
    number | undefined
  >();
  const [fields, setFields] = useState<DatabaseField[]>([]);
  const [originalFields, setOriginalFields] = useState<DatabaseField[]>([]); // Track original fields from database
  const [sections, setSections] = useState<FormSection[]>(DEFAULT_SECTIONS);
  const [expandedSections, setExpandedSections] = useState<Set<string>>(
    () => new Set(["section_logic_reasoning"]),
  );
  const [openSectionMenuId, setOpenSectionMenuId] = useState<string | null>(null);
  const [savedForms, setSavedForms] = useState<SavedForm[]>([]);
  const [selectedForm, setSelectedForm] = useState<SavedForm | null>(null);
  const [formResponses, setFormResponses] = useState<FormResponse>({});

  const [showSubmitSuccess, setShowSubmitSuccess] = useState(false);
  const [errors, setErrors] = useState<{ [key: string]: string }>({});
  const [isSaving, setIsSaving] = useState(false);
  const [saveErrors, setSaveErrors] = useState<string[]>([]);
  const [fieldErrors, setFieldErrors] = useState<{
    [fieldId: string]: string[];
  }>({});
  const [clickedFieldType, setClickedFieldType] = useState<string | null>(null);
  const [selectedFieldType, setSelectedFieldType] = useState<FieldType>("text");
  const [showLinkCopied, setShowLinkCopied] = useState(false);
  const [showAddSectionModal, setShowAddSectionModal] = useState(false);
  const [newSectionTitle, setNewSectionTitle] = useState("");
  const [newSectionType, setNewSectionType] = useState<SectionType>("custom");
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const [isFormSaved, setIsFormSaved] = useState(false);
  const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false);
  const [focusedOptionId, setFocusedOptionId] = useState<string | null>(null);

  // DnD sensors — require small movement before drag so button clicks work
  const sensors = useSensors(
    useSensor(PointerSensor, {
      activationConstraint: { distance: 8 },
    }),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    }),
  );

  // Calculate total score
  const totalScore = fields.reduce((sum, field) => sum + (field.score || 0), 0);

  // Section management functions
  const getSectionFields = (sectionId: string) => {
    return fields.filter((field) => field.sectionId === sectionId);
  };

  const getSectionQuestionCount = (sectionId: string) => {
    return getSectionFields(sectionId).length;
  };

  const toggleSectionExpanded = (sectionId: string) => {
    setExpandedSections((prev) => {
      const next = new Set(prev);
      if (next.has(sectionId)) {
        next.delete(sectionId);
      } else {
        next.add(sectionId);
      }
      return next;
    });
  };

  const isSectionExpanded = (sectionId: string) => expandedSections.has(sectionId);

  const defaultMcOptions = (): Option[] => [
    { id: `${Date.now()}-a`, value: "Option A" },
    { id: `${Date.now()}-b`, value: "Option B" },
  ];

  const addFieldToSection = (sectionId: string, type: FieldType = "multipleChoice") => {
    const newField: DatabaseField = {
      id: Date.now().toString(),
      type,
      sectionId,
      label: "",
      required: true,
      answer: "",
      score: 0,
      options:
        type === "multipleChoice" || type === "checkboxes" || type === "dropdown"
          ? defaultMcOptions()
          : undefined,
    };
    setFields([...fields, newField]);
    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const duplicateSection = (sectionId: string) => {
    const section = sections.find((s) => s.id === sectionId);
    if (!section) return;

    const newSectionId = `section_${Date.now()}`;
    const newSection: FormSection = {
      ...section,
      id: newSectionId,
      title: `${section.title}`,
      order: sections.length + 1,
    };

    const sectionFields = fields.filter((f) => f.sectionId === sectionId);
    const duplicatedFields = sectionFields.map((field) => ({
      ...field,
      id: `${Date.now()}_${Math.random().toString(36).slice(2)}`,
      sectionId: newSectionId,
      dbId: undefined,
      options: field.options?.map((opt) => ({
        ...opt,
        id: `${Date.now()}-${opt.id}`,
      })),
    }));

    setSections([...sections, newSection]);
    setFields([...fields, ...duplicatedFields]);
    setExpandedSections((prev) => new Set([...Array.from(prev), newSectionId]));
    setOpenSectionMenuId(null);
    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const updateSectionTitle = (sectionId: string, title: string) => {
    setSections(sections.map(s => s.id === sectionId ? { ...s, title } : s));
    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const updateSectionDescription = (sectionId: string, description: string) => {
    setSections(sections.map(s => s.id === sectionId ? { ...s, description } : s));
    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const getSectionScore = (sectionId: string) => {
    return fields
      .filter(f => f.sectionId === sectionId)
      .reduce((sum, field) => sum + (field.score || 0), 0);
  };

  const addNewSection = () => {
    if (!newSectionTitle.trim()) {
      alert("Please enter a section title");
      return;
    }

    const newSection: FormSection = {
      id: `section_${Date.now()}`,
      type: newSectionType,
      title: newSectionTitle,
      description: "",
      order: sections.length + 1,
      fields: [],
    };

    setSections([...sections, newSection]);
    setExpandedSections((prev) => new Set([...Array.from(prev), newSection.id]));
    setNewSectionTitle("");
    setNewSectionType("custom");
    setShowAddSectionModal(false);
    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const deleteSection = (sectionId: string) => {
    if (sections.length <= 1) {
      alert("You must have at least one section");
      return;
    }

    // Remove fields in this section
    setFields(fields.filter(f => f.sectionId !== sectionId));
    
    // Remove section
    const updatedSections = sections.filter(s => s.id !== sectionId);
    setSections(updatedSections);
    
    setExpandedSections((prev) => {
      const next = new Set(prev);
      next.delete(sectionId);
      if (next.size === 0 && updatedSections[0]) {
        next.add(updatedSections[0].id);
      }
      return next;
    });
    setOpenSectionMenuId(null);

    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  // Close section menu when clicking outside
  useEffect(() => {
    if (!openSectionMenuId) return;
    const handleClick = () => setOpenSectionMenuId(null);
    document.addEventListener("click", handleClick);
    return () => document.removeEventListener("click", handleClick);
  }, [openSectionMenuId]);

  // Load form from database if id is provided
  useEffect(() => {
    const urlParams = new URLSearchParams(window.location.search);
    const mode = urlParams.get("mode");
    const from = urlParams.get("from");
    if (mode === "preview") {
      setIsPreviewMode(true);
      setViewMode("preview");
    }

    if (id && tokens?.accessToken) {
      setIsLoading(true);
      fetchFormFromDatabase(id);
    } else if (id && !tokens?.accessToken) {
      // Wait for authentication tokens to load
      setIsLoading(true);
    } else {
      setIsLoading(false);
    }
  }, [id, tokens?.accessToken]);

  const fetchFormFromDatabase = async (formId: string) => {
    try {
      const graphqlClient = new GraphQLClient(
        config.api.baseUrl + "/api/graphql",
        tokens?.accessToken,
      );

      const query = `
        query MyQuery($formId: String = "") {
          AptitudeTestTemplate(where: {id: {_eq: $formId}}) {
            id
            title
            timeLimitMinutes
            totalScore
            createdAt
            AptitudeTestQuestions(order_by: {order: asc}) {
              answer
              fieldType
              questionId: id
              label
              options
              order
              score
            }
          }
        }
      `;

      const variables = { formId };
      const response = await graphqlClient.query(query, variables);

      const template = response?.AptitudeTestTemplate?.[0];
      if (template) {
        setFormTitle(template.title);
        setFormDescription(""); // No description in query
        setTimeLimitMinutes(template.timeLimitMinutes);

        // Map questions to DatabaseField format
        const dbFields: DatabaseField[] = template.AptitudeTestQuestions.map(
          (question: any) => {
            // Map API fieldType back to component type
            let componentType: FieldType;
            switch (question.fieldType) {
              case "multiple_choice":
                componentType = "multipleChoice";
                break;
              case "checkbox":
                componentType = "checkboxes";
                break;
              default:
                componentType = question.fieldType as FieldType;
            }

            // Map options from array of strings to {id, value}[]
            const options: Option[] = question.options
              ? question.options.map((opt: string, index: number) => ({
                  id: index.toString(),
                  value: opt,
                }))
              : [];

            // Handle answer based on fieldType
            let answer: string = question.answer || "";
            if (
              componentType === "checkboxes" &&
              Array.isArray(question.answer)
            ) {
              answer = question.answer.join(",");
            }

            return {
              id: question.questionId,
              type: componentType,
              sectionId: "section_logic_reasoning",
              label: question.label,
              required: true, // All test questions are required
              options,
              answer,
              score: question.score,
              dbId: question.questionId,
            };
          },
        );

        // Sort fields by order
        dbFields.sort((a, b) => {
          const orderA =
            template.AptitudeTestQuestions.find(
              (q: any) => q.questionId === a.id,
            )?.order || 0;
          const orderB =
            template.AptitudeTestQuestions.find(
              (q: any) => q.questionId === b.id,
            )?.order || 0;
          return orderA - orderB;
        });

        setFields(dbFields);
        setOriginalFields([...dbFields]);
      } else {
        // If no template found, set default
        setFormTitle("Aptitude Test Form");
        setFormDescription("");
        setTimeLimitMinutes(undefined);
        setOriginalFields([]);
      }
    } catch (error) {
      console.error("Failed to load form from database:", error);
      setFormTitle("Aptitude Test Form");
      setFormDescription("");
      setTimeLimitMinutes(undefined);
      setOriginalFields([]);
    } finally {
      setIsLoading(false);
    }
  };

  const Icon123 = (props: any) => (
    <span
      className={`material-symbols-outlined ${props.className || ""}`}
      style={{ fontSize: props.size || 20 }}
    >
      123
    </span>
  );

  const addField = (sectionId: string, type: FieldType = "multipleChoice") => {
    setClickedFieldType(type);
    setTimeout(() => setClickedFieldType(null), 150);
    setSelectedFieldType(type);
    addFieldToSection(sectionId, type);

    if (isSidebarOpen) {
      setIsSidebarOpen(false);
    }
  };

  const updateField = (id: string, updates: Partial<DatabaseField>) => {
    setFields(
      fields.map((field) =>
        field.id === id ? { ...field, ...updates } : field,
      ),
    );
    // Clear field errors when user starts editing
    if (fieldErrors[id]) {
      setFieldErrors((prev) => {
        const newErrors = { ...prev };
        delete newErrors[id];
        return newErrors;
      });
    }
    // Mark as having unsaved changes
    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const deleteField = (id: string) => {
    setFields((prev) => prev.filter((field) => field.id !== id));

    setFieldErrors((prev) => {
      if (!prev[id]) return prev;
      const next = { ...prev };
      delete next[id];
      return next;
    });

    setHasUnsavedChanges(true);
    setIsFormSaved(false);
  };

  const handleDeleteQuestion = (
    e: React.MouseEvent,
    fieldId: string,
  ) => {
    e.stopPropagation();
    e.preventDefault();
    deleteField(fieldId);
  };

  const duplicateField = (id: string) => {
    const fieldToDuplicate = fields.find((field) => field.id === id);
    if (fieldToDuplicate) {
      const duplicatedOptions = fieldToDuplicate.options?.map((option) => ({
        ...option,
        id: `${Date.now()}-${option.id}`,
      }));

      const newField = {
        ...fieldToDuplicate,
        id: Date.now().toString(),
        answer: "",
        options: duplicatedOptions,
        dbId: undefined, // Ensure it's treated as a new field
      };
      const index = fields.findIndex((field) => field.id === id);
      const newFields = [...fields];
      newFields.splice(index + 1, 0, newField);
      setFields(newFields);
      // Mark as having unsaved changes
      setHasUnsavedChanges(true);
      setIsFormSaved(false);
    }
  };

  const addOption = (fieldId: string) => {
    const field = fields.find((f) => f.id === fieldId);
    if (field && field.options) {
      const newOptionId = Date.now().toString();
      const newOption: Option = {
        id: newOptionId,
        value: "",
      };
      updateField(fieldId, {
        options: [...field.options, newOption],
      });
      // Set the new option to be focused
      setFocusedOptionId(newOptionId);
    }
  };

  const updateOption = (fieldId: string, optionId: string, value: string) => {
    const field = fields.find((f) => f.id === fieldId);
    if (field && field.options) {
      updateField(fieldId, {
        options: field.options.map((opt) =>
          opt.id === optionId ? { ...opt, value } : opt,
        ),
      });
    }
  };

  const deleteOption = (fieldId: string, optionId: string) => {
    const field = fields.find((f) => f.id === fieldId);
    if (field && field.options && field.options.length > 1) {
      updateField(fieldId, {
        options: field.options.filter((opt) => opt.id !== optionId),
      });
    }
  };

  // Drag and drop handler
  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event;

    if (over && active.id !== over.id) {
      setFields((items) => {
        const oldIndex = items.findIndex((item) => item.id === active.id);
        const newIndex = items.findIndex((item) => item.id === over.id);

        return arrayMove(items, oldIndex, newIndex);
      });
    }
  };


  const router = useRouter();
  const saveForm = async () => {
    if (!tokens?.accessToken) {
      setSaveErrors(["Authentication token is required"]);
      return;
    }

    // Clear prior field errors before running validation.
    setFieldErrors({});

    // Validate time limit is mandatory
    if (!timeLimitMinutes || timeLimitMinutes <= 0) {
      setSaveErrors(["Time limit is required and must be greater than 0"]);
      return;
    }

    // Validate that all questions have labels
    const questionsWithoutLabels = fields.filter(
      (field) => !field.label || field.label.trim() === "",
    );
    if (questionsWithoutLabels.length > 0) {
      setSaveErrors(["Please check your form fields and try again"]);
      return;
    }

    // Validate that all questions have answers
    const questionsWithoutAnswers = fields.filter(
      (field) => !field.answer || field.answer.trim() === "",
    );
    if (questionsWithoutAnswers.length > 0) {
      setSaveErrors([
        "All questions must have answers. Please provide answers for all questions.",
      ]);
      return;
    }

    // Validate that all questions have scores
    const questionsWithoutScores = fields.filter(
      (field) =>
        field.score === undefined || field.score === null || field.score <= 0,
    );
    if (questionsWithoutScores.length > 0) {
      const fieldErrorMap: { [fieldId: string]: string[] } = {};
      const userFriendlyErrors: string[] = [];

      questionsWithoutScores.forEach((field) => {
        const questionIndex = fields.findIndex((f) => f.id === field.id) + 1;
        fieldErrorMap[field.id] = [" "];
        userFriendlyErrors.push(
          `Question ${questionIndex}: Score must be greater than 0`,
        );
      });

      setFieldErrors(fieldErrorMap);
      setSaveErrors(userFriendlyErrors);
      return;
    }

    // Validate options for multiple choice, checkboxes, and dropdown fields
    const fieldsWithEmptyOptions: {
      field: DatabaseField;
      emptyOptionIndices: number[];
    }[] = [];

    fields.forEach((field) => {
      if (["multipleChoice", "checkboxes", "dropdown"].includes(field.type)) {
        if (field.options && field.options.length > 0) {
          const emptyOptionIndices: number[] = [];
          field.options.forEach((option, index) => {
            if (!option.value || option.value.trim() === "") {
              emptyOptionIndices.push(index + 1); // 1-based indexing
            }
          });
          if (emptyOptionIndices.length > 0) {
            fieldsWithEmptyOptions.push({ field, emptyOptionIndices });
          }
        }
      }
    });

    if (fieldsWithEmptyOptions.length > 0) {
      const fieldErrorMap: { [fieldId: string]: string[] } = {};
      const userFriendlyErrors: string[] = [];

      fieldsWithEmptyOptions.forEach(({ field, emptyOptionIndices }) => {
        const questionIndex = fields.findIndex((f) => f.id === field.id) + 1;
        emptyOptionIndices.forEach((optionIndex) => {
          userFriendlyErrors.push(
            `Question ${questionIndex}: Option ${optionIndex} cannot be empty`,
          );
        });
        fieldErrorMap[field.id] = emptyOptionIndices.map(
          (optionIndex) => `Option ${optionIndex} cannot be empty`,
        );
      });

      setFieldErrors(fieldErrorMap);
      setSaveErrors(userFriendlyErrors);
      return;
    }

    // If this is a new test (id is 'new' or undefined), create in DB on first save
    if (!id || id === "new") {
      setIsSaving(true);
      setSaveErrors([]);
      try {
        // Create the test template in the DB
        const response = await createAptitudeTestTemplate(
          formTitle,
          formDescription,
          timeLimitMinutes,
          totalScore,
          tokens.accessToken
        );
        const newId = response?.data?.id || response?.id || response?.template?.id;
        if (newId) {
          // Immediately save the current questions to the new test
          const questions = fields.map((field, index) => {
            let apiType: string = field.type;
            let apiOptions = undefined;
            let apiAnswer: string | string[] | undefined = field.answer;
            switch (field.type) {
              case "longText":
                apiType = "text";
                break;
              case "multipleChoice":
                apiType = "multiple_choice";
                break;
              case "checkboxes":
                apiType = "checkbox";
                break;
              case "fileUpload":
                return null;
              default:
                apiType = field.type;
            }
            if (field.type === "checkboxes" && field.answer) {
              apiAnswer = field.answer.split(",").filter((a) => a.trim());
            } else if (["multipleChoice", "dropdown"].includes(field.type)) {
              apiAnswer = field.answer;
            }
            if (field.options && ["multiple_choice", "checkbox", "dropdown"].includes(apiType)) {
              apiOptions = field.options.map((opt) => opt.value);
            }
            return {
              fieldType: apiType,
              label: field.label,
              ...(apiOptions ? { options: apiOptions } : {}),
              ...(apiAnswer ? { answer: apiAnswer } : {}),
              ...(field.score ? { score: field.score } : {}),
              order: index + 1,
            };
          }).filter(Boolean);
          await updateAptitudeTestTemplate(
            newId,
            formTitle,
            formDescription,
            timeLimitMinutes,
            totalScore,
            questions,
            tokens.accessToken
          );
          // Now update the URL to use the new id, replace so no back navigation
          if (router) router.replace(`/test-form/${newId}`);
        } else {
          throw new Error("Failed to create test template");
        }
      } catch (error) {
        let msg = "Failed to create test";
        if (error instanceof Error) {
          msg = error.message;
        } else if (typeof error === "string") {
          msg = error;
        }
        setSaveErrors([msg]);
      } finally {
        setIsSaving(false);
      }
      return;
    }
    if (!id) {
      setSaveErrors(["Form ID is required"]);
      return;
    }

    setIsSaving(true);
    setSaveErrors([]);
    try {
      // Map fields to questions format for API
      const questions: any[] = [];

      fields.forEach((field, index) => {
        let apiType: string = field.type;
        let apiOptions = undefined;
        let apiAnswer: any = field.answer;

        // Map field types to API expected types
        switch (field.type) {
          case "longText":
            // long text is stored as plain text on the API
            apiType = "text";
            break;
          case "multipleChoice":
            apiType = "multiple_choice";
            break;
          case "checkboxes":
            apiType = "checkbox";
            break;
          case "fileUpload":
            // Skip file upload fields as they're not supported by the API
            return;
          default:
            apiType = field.type;
        }

        // Handle answer arrays for different field types
        if (field.type === "checkboxes" && field.answer) {
          // For checkboxes, answer is stored as comma-separated string
          apiAnswer = field.answer.split(",").filter((a) => a.trim());
        } else if (
          field.type === "multipleChoice" ||
          field.type === "dropdown"
        ) {
          // For single choice, keep as string
          apiAnswer = field.answer;
        } else {
          apiAnswer = field.answer;
        }

        // Only include options for supported field types
        if (
          field.options &&
          ["multiple_choice", "checkbox", "dropdown"].includes(apiType)
        ) {
          apiOptions = field.options.map((opt) => opt.value);
        }

        const question: any = {
          fieldType: apiType,
          label: field.label, // Multiline text with newlines is preserved as-is
          ...(apiOptions ? { options: apiOptions } : {}),
          ...(apiAnswer ? { answer: apiAnswer } : {}),
          ...(field.score ? { score: field.score } : {}),
          order: index + 1,
        };
        questions.push(question);
      });

      const result = await updateAptitudeTestTemplate(
        id,
        formTitle,
        formDescription,
        timeLimitMinutes,
        totalScore,
        questions,
        tokens.accessToken,
      );

      console.log("Test saved to database:", result);

      // Update fields with database IDs from the response if available
      if (result && result.data && result.data.questions) {
        const updatedFields = [...fields];
        const questionsFromResponse = result.data.questions;

        // Map created questions back to fields by order
        questionsFromResponse.forEach(
          (question: { id: string }, index: number) => {
            if (updatedFields[index] && !updatedFields[index].dbId) {
              updatedFields[index] = {
                ...updatedFields[index],
                dbId: question.id,
              };
            }
          },
        );

        setFields(updatedFields);
        setOriginalFields([...updatedFields]);
      } else {
        // Update original fields to reflect current state
        setOriginalFields([...fields]);
      }
      // Mark form as saved and show success message
      setIsFormSaved(true);
      setHasUnsavedChanges(false);
      toast.success("Test saved successfully!");
      setTimeout(() => setShowSubmitSuccess(false), 3000);

      // Don't navigate back - keep the form open
    } catch (error: any) {
      console.error("Failed to save test:", error);

      // Try to extract backend error messages
      let errorMessages: string[] = [];

      if (error.response?.data?.errors) {
        // Handle array of error strings
        if (Array.isArray(error.response.data.errors)) {
          errorMessages = error.response.data.errors;
        } else if (typeof error.response.data.errors === "string") {
          errorMessages = [error.response.data.errors];
        }
      } else if (error.response?.data?.message) {
        // Handle single message
        errorMessages = [error.response.data.message];
      } else if (error.response?.data) {
        // Handle other error formats
        const data = error.response.data;
        if (typeof data === "string") {
          errorMessages = [data];
        } else if (data.error) {
          errorMessages = [data.error];
        } else {
          // Try to extract any string values from the error object
          const messages: string[] = [];
          Object.values(data).forEach((value) => {
            if (typeof value === "string") {
              messages.push(value);
            } else if (Array.isArray(value)) {
              value.forEach((v) => {
                if (typeof v === "string") messages.push(v);
              });
            }
          });
          errorMessages =
            messages.length > 0
              ? messages
              : ["An error occurred while saving the test"];
        }
      } else if (error.message) {
        // Fallback to error message
        errorMessages = [error.message];
      } else {
        errorMessages = ["An error occurred while saving the test"];
      }

      // Transform error messages to be user-friendly
      const userFriendlyErrors = errorMessages.map((errorMsg: string) => {
        // Parse errors like: "\"questions[0].label\" length must be at least 2 characters long"
        // or "\"questions[2].options[1]\" is not allowed to be empty"
        const match = errorMsg.match(/"questions\[(\d+)\]\.(.+)" (.+)/);
        if (match) {
          const [, indexStr, fieldProp, errorText] = match;
          const index = parseInt(indexStr) + 1; // Convert to 1-based indexing for user display

          if (fieldProp === "label") {
            if (errorText.includes("length must be at least")) {
              return `Question ${index}: Label must be at least 2 characters long`;
            } else if (errorText.includes("is not allowed to be empty")) {
              return `Question ${index}: Label cannot be empty`;
            }
          } else if (fieldProp.startsWith("options[")) {
            // Parse option errors like: "options[1]" is not allowed to be empty
            const optionMatch = fieldProp.match(/options\[(\d+)\]/);
            if (optionMatch) {
              const optionIndex = parseInt(optionMatch[1]) + 1;
              if (errorText.includes("is not allowed to be empty")) {
                return `Question ${index}: Option ${optionIndex} cannot be empty`;
              }
            }
          } else if (fieldProp === "answer") {
            if (errorText.includes("is not allowed to be empty")) {
              return `Question ${index}: Answer cannot be empty`;
            }
          } else if (fieldProp === "score") {
            if (errorText.includes("must be a number")) {
              return `Question ${index}: Score must be a valid number`;
            } else if (errorText.includes("must be larger than or equal to")) {
              return `Question ${index}: Score must be greater than 0`;
            }
          }
        }

        // Handle database constraint errors
        if (
          errorMsg.includes("value too long for type character varying(255)")
        ) {
          return "Label is too long (maximum 255 characters)";
        }

        // If we can't parse it, return a generic message
        return "Please check your form fields and try again";
      });

      setSaveErrors(userFriendlyErrors);

      // Parse field-specific errors
      const fieldErrorMap: { [fieldId: string]: string[] } = {};
      errorMessages.forEach((errorMsg: string) => {
        // Parse errors like: "\"questions[0].label\" length must be at least 2 characters long"
        const match = errorMsg.match(/"questions\[(\d+)\]\.(\w+)" (.+)/);
        if (match) {
          const [, indexStr, fieldProp, errorText] = match;
          const index = parseInt(indexStr);
          const field = fields[index];
          if (field) {
            if (!fieldErrorMap[field.id]) {
              fieldErrorMap[field.id] = [];
            }

            // Convert to user-friendly field-specific errors
            if (fieldProp === "label") {
              if (errorText.includes("length must be at least")) {
                fieldErrorMap[field.id].push(
                  "Label must be at least 2 characters long",
                );
              } else if (errorText.includes("is not allowed to be empty")) {
                fieldErrorMap[field.id].push("Label cannot be empty");
              } else {
                fieldErrorMap[field.id].push(`Label: ${errorText}`);
              }
            } else if (fieldProp === "options") {
              const optionMatch = errorText.match(/options\[(\d+)\]" (.+)/);
              if (optionMatch) {
                const [, optionIndexStr, optionError] = optionMatch;
                const optionIndex = parseInt(optionIndexStr) + 1;
                if (optionError.includes("is not allowed to be empty")) {
                  fieldErrorMap[field.id].push(
                    `Option ${optionIndex} cannot be empty`,
                  );
                } else {
                  fieldErrorMap[field.id].push(
                    `Option ${optionIndex}: ${optionError}`,
                  );
                }
              }
            } else if (fieldProp === "answer") {
              if (errorText.includes("is not allowed to be empty")) {
                fieldErrorMap[field.id].push("Answer cannot be empty");
              } else {
                fieldErrorMap[field.id].push(`Answer: ${errorText}`);
              }
            } else if (fieldProp === "score") {
              if (errorText.includes("must be a number")) {
                fieldErrorMap[field.id].push("Score must be a valid number");
              } else if (
                errorText.includes("must be larger than or equal to")
              ) {
                fieldErrorMap[field.id].push("Score must be greater than 0");
              } else {
                fieldErrorMap[field.id].push(`Score: ${errorText}`);
              }
            } else {
              fieldErrorMap[field.id].push(`${fieldProp}: ${errorText}`);
            }
          }
        }
      });
      setFieldErrors(fieldErrorMap);
    } finally {
      setIsSaving(false);
    }
  };

  const openForm = (form: SavedForm) => {
    setSelectedForm(form);
    setFormResponses({});
    setErrors({});
    setViewMode("fillForm");
  };

  const handleResponseChange = (fieldId: string, value: any) => {
    setFormResponses({
      ...formResponses,
      [fieldId]: value,
    });
    // Clear error when user starts typing
    if (errors[fieldId]) {
      setErrors({ ...errors, [fieldId]: "" });
    }
  };

  const handleCheckboxChange = (
    fieldId: string,
    optionValue: string,
    checked: boolean,
  ) => {
    const currentValues = formResponses[fieldId] || [];
    const newValues = checked
      ? [...currentValues, optionValue]
      : currentValues.filter((v: string) => v !== optionValue);
    handleResponseChange(fieldId, newValues);
  };

  const validateForm = (): boolean => {
    const newErrors: { [key: string]: string } = {};

    selectedForm?.fields.forEach((field) => {
      if (field.required) {
        const value = formResponses[field.id];
        if (
          !value ||
          (Array.isArray(value) && value.length === 0) ||
          value === ""
        ) {
          newErrors[field.id] = "This field is required";
        }
      }
    });

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = () => {
    if (validateForm()) {
      console.log("Form submitted:", {
        formId: selectedForm?.id,
        formTitle: selectedForm?.title,
        responses: formResponses,
      });
      setShowSubmitSuccess(true);
      setTimeout(() => {
        setShowSubmitSuccess(false);
        setViewMode("builder");
        setSelectedForm(null);
        setFormResponses({});
      }, 2000);
    }
  };

  const copyFormLink = async () => {
    if (!id) {
      console.warn("No form ID available to copy link");
      return;
    }

    const formUrl = `${window.location.origin}/test-form/${id}`;

    try {
      await navigator.clipboard.writeText(formUrl);
      setShowLinkCopied(true);
      setTimeout(() => setShowLinkCopied(false), 3000);
    } catch (error) {
      console.error("Failed to copy link to clipboard:", error);
      // Fallback for older browsers
      const textArea = document.createElement("textarea");
      textArea.value = formUrl;
      document.body.appendChild(textArea);
      textArea.select();
      try {
        document.execCommand("copy");
        setShowLinkCopied(true);
        setTimeout(() => setShowLinkCopied(false), 3000);
      } catch (fallbackError) {
        console.error("Fallback copy failed:", fallbackError);
      }
      document.body.removeChild(textArea);
    }
  };

  const renderFieldEditor = (field: FormField, questionIndex: number) => {
    const labelHasError = fieldErrors[field.id]?.some((error) => error.includes("label"));
    const scoreHasError = fieldErrors[field.id]?.some((error) => error.includes("Score"));
    const answerHasError = fieldErrors[field.id]?.some((error) => error.includes("Answer"));

    return (
      <div className="bg-white rounded-lg border border-gray-200 p-4 sm:p-5 mb-4">
        <div className="flex items-center justify-between mb-4">
          <span className="text-sm font-medium text-gray-600">
            Question - {questionIndex}
          </span>
          <div className="flex items-center gap-2">
            <span className="text-sm text-gray-600">Score</span>
            <input
              type="number"
              min="0"
              max="100"
              step="0.5"
              value={field.score ?? ""}
              onChange={(e) => {
                const value = e.target.value;
                const numValue = parseFloat(value);
                if (value === "" || (numValue >= 0 && numValue <= 100)) {
                  updateField(field.id, {
                    score: value === "" ? undefined : numValue,
                  });
                }
              }}
              className={`w-16 px-2 py-1.5 border rounded-md text-center text-sm focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                scoreHasError ? "border-red-500" : "border-gray-300"
              }`}
              placeholder="0"
            />
          </div>
        </div>

        <input
          type="text"
          value={field.label}
          onChange={(e) => updateField(field.id, { label: e.target.value })}
          className={`w-full px-3 py-2.5 border rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-purple-500 mb-4 ${
            labelHasError
              ? "border-red-500"
              : "border-gray-300 focus:border-purple-500"
          }`}
          placeholder="Enter the Question"
        />
        {labelHasError && (
          <div className="text-red-500 text-sm -mt-2 mb-3">
            {fieldErrors[field.id]
              ?.filter((error) => error.includes("label"))
              .join(", ")}
          </div>
        )}

        {["multipleChoice", "checkboxes", "dropdown"].includes(field.type) &&
          field.options && (
            <div className="space-y-2 mb-3">
              {field.options.map((option) => (
                <div key={option.id} className="flex items-center gap-2">
                  {field.type === "multipleChoice" ? (
                    <CircleDot size={18} className="text-gray-400 shrink-0" />
                  ) : (
                    <CheckSquare size={18} className="text-gray-400 shrink-0" />
                  )}
                  <input
                    type="text"
                    value={option.value}
                    onChange={(e) =>
                      updateOption(field.id, option.id, e.target.value)
                    }
                    autoFocus={focusedOptionId === option.id}
                    onFocus={() => setFocusedOptionId(null)}
                    className="flex-1 px-3 py-2 border border-gray-200 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-purple-500"
                    placeholder="Option"
                  />
                  {field.options && field.options.length > 1 && (
                    <button
                      type="button"
                      onPointerDown={(e) => e.stopPropagation()}
                      onClick={(e) => {
                        e.stopPropagation();
                        deleteOption(field.id, option.id);
                      }}
                      className="text-gray-400 hover:text-red-500 p-1"
                      aria-label="Remove option"
                    >
                      <X size={16} />
                    </button>
                  )}
                </div>
              ))}
              {fieldErrors[field.id]?.some((error) => error.includes("Option")) && (
                <div className="text-red-500 text-sm">
                  {fieldErrors[field.id]
                    ?.filter((error) => error.includes("Option"))
                    .join(", ")}
                </div>
              )}
              <button
                type="button"
                onClick={() => addOption(field.id)}
                className="text-sm text-[#7c3aed] hover:text-[#6d28d9] font-medium flex items-center gap-1 mt-1"
              >
                <Plus size={16} />
                Add Option
              </button>
            </div>
          )}

        {field.type === "fileUpload" && (
          <div className="border-2 border-dashed border-gray-300 rounded-lg p-4 text-center text-gray-400 text-sm mb-4">
            <div className="flex items-center justify-center gap-2">
              <Upload size={16} className="text-gray-600" />
              <span>Add File</span>
            </div>
          </div>
        )}

        <div className="flex items-end justify-between gap-4 mt-2">
          <div className="flex items-center gap-3 flex-1 min-w-0">
            <span className="text-sm font-medium text-gray-700 shrink-0">Answer</span>
            <div className="flex-1 max-w-xs">
              {field.type === "text" ||
              field.type === "longText" ||
              field.type === "email" ||
              field.type === "number" ? (
                <input
                  type={
                    field.type === "number"
                      ? "number"
                      : field.type === "email"
                        ? "email"
                        : "text"
                  }
                  value={field.answer || ""}
                  onChange={(e) =>
                    updateField(field.id, { answer: e.target.value })
                  }
                  className={`w-full px-3 py-2 border rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                    answerHasError ? "border-red-500" : "border-gray-300"
                  }`}
                  placeholder="Enter the correct answer"
                />
              ) : field.type === "multipleChoice" || field.type === "dropdown" ? (
                <div className="relative">
                  <select
                    value={field.answer || ""}
                    onChange={(e) =>
                      updateField(field.id, { answer: e.target.value })
                    }
                    className={`w-full px-3 py-2 pr-10 border rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-purple-500 appearance-none bg-white ${
                      answerHasError ? "border-red-500" : "border-gray-300"
                    }`}
                  >
                    <option value="">Select answer</option>
                    {field.options?.map((option) => (
                      <option key={option.id} value={option.value}>
                        {option.value || "—"}
                      </option>
                    ))}
                  </select>
                  <ChevronDown className="absolute right-3 top-1/2 -translate-y-1/2 w-4 h-4 text-gray-400 pointer-events-none" />
                </div>
              ) : field.type === "checkboxes" ? (
                <div className="space-y-2">
                  {field.options?.map((option) => (
                    <label
                      key={option.id}
                      className="flex items-center gap-2 text-sm text-gray-700"
                    >
                      <input
                        type="checkbox"
                        checked={(field.answer || "")
                          .split(",")
                          .includes(option.value)}
                        onChange={(e) => {
                          const currentAnswers = (field.answer || "")
                            .split(",")
                            .filter((a) => a);
                          const newAnswers = e.target.checked
                            ? [...currentAnswers, option.value]
                            : currentAnswers.filter((a) => a !== option.value);
                          updateField(field.id, {
                            answer: newAnswers.join(","),
                          });
                        }}
                        className="w-4 h-4"
                        style={{ accentColor: "#7c3aed" }}
                      />
                      {option.value}
                    </label>
                  ))}
                </div>
              ) : null}
              {answerHasError && (
                <div className="text-red-500 text-xs mt-1">
                  {fieldErrors[field.id]
                    ?.filter((error) => error.includes("Answer"))
                    .join(", ")}
                </div>
              )}
            </div>
          </div>

          <div className="flex items-center gap-1 shrink-0">
            <TooltipProvider delayDuration={150}>
              <Tooltip>
                <TooltipTrigger asChild>
                  <button
                    type="button"
                    onPointerDown={(e) => e.stopPropagation()}
                    onClick={(e) => {
                      e.stopPropagation();
                      duplicateField(field.id);
                    }}
                    className="text-gray-400 hover:text-gray-700 p-2"
                  >
                    <Copy size={18} />
                  </button>
                </TooltipTrigger>
                <TooltipContent className="text-xs p-2 bg-white rounded shadow-md">
                  Duplicate
                </TooltipContent>
              </Tooltip>
              <Tooltip>
                <TooltipTrigger asChild>
                  <button
                    type="button"
                    onPointerDown={(e) => e.stopPropagation()}
                    onClick={(e) => handleDeleteQuestion(e, field.id)}
                    className="text-red-400 hover:text-red-600 p-2"
                    aria-label="Delete question"
                  >
                    <Trash2 size={18} />
                  </button>
                </TooltipTrigger>
                <TooltipContent className="text-xs p-2 bg-white rounded shadow-md">
                  Delete
                </TooltipContent>
              </Tooltip>
            </TooltipProvider>
          </div>
        </div>
      </div>
    );
  };

  const renderFieldPreview = (field: FormField) => {
    return (
      <div
        key={field.id}
        className="bg-white rounded-lg border border-gray-200 p-4 sm:p-6 mb-4"
      >
        <label className="block text-base font-medium text-gray-900 mb-3 break-words whitespace-pre-wrap">
          {field.label}
          {field.required && <span className="text-red-500">*</span>}
        </label>

        {field.type === "text" && (
          <input
            type="text"
            placeholder="Text"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500"
            disabled
          />
        )}

        {field.type === "longText" && (
          <textarea
            placeholder="Long Text"
            rows={4}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500"
            disabled
          />
        )}

        {field.type === "email" && (
          <input
            type="email"
            placeholder="Email"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500"
            disabled
          />
        )}

        {field.type === "number" && (
          <input
            type="number"
            placeholder="Number"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500"
            disabled
          />
        )}

        {field.type === "multipleChoice" && field.options && (
          <div className="space-y-2">
            {field.options.map((option) => (
              <label
                key={option.id}
                className="flex items-center gap-2 cursor-pointer"
              >
                <input
                  type="radio"
                  name={field.id}
                  className="w-4 h-4"
                  disabled
                />
                <span className="text-gray-700">{option.value}</span>
              </label>
            ))}
          </div>
        )}

        {field.type === "checkboxes" && field.options && (
          <div className="space-y-2">
            {field.options.map((option) => (
              <label
                key={option.id}
                className="flex items-center gap-2 cursor-pointer"
              >
                <input
                  type="checkbox"
                  className="w-4 h-4"
                  style={{ accentColor: "#7c3ead" }}
                  disabled
                />
                <span className="text-gray-700">{option.value}</span>
              </label>
            ))}
          </div>
        )}

        {field.type === "dropdown" && field.options && (
          <select
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500"
            disabled
          >
            <option>Select an option</option>
            {field.options.map((option) => (
              <option key={option.id}>{option.value}</option>
            ))}
          </select>
        )}

        {field.type === "fileUpload" && (
          <div className="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center">
            <button className="px-4 py-2 text-purple-600 border border-purple-600 rounded-md hover:bg-purple-50 flex items-center gap-2 mx-auto">
              <Upload size={16} className="text-purple-600" />
              <span>Add File</span>
            </button>
          </div>
        )}

        {/* Show Answer and Score */}
        {(field.answer || field.score) && (
          <div className="mt-4 p-3 bg-green-50 border border-green-200 rounded-md">
            <div className="space-y-2">
              {field.answer && (
                <div className="flex items-start gap-2">
                  <p className="text-sm font-medium text-green-800 whitespace-nowrap">
                    Correct Answer:
                  </p>
                  <p className="text-sm text-green-700 break-words truncate">
                    {field.answer}
                  </p>
                </div>
              )}
              {field.score && (
                <div className="flex items-center gap-2">
                  <p className="text-sm font-medium text-green-800">Score:</p>
                  <p className="text-sm text-green-700">{field.score}</p>
                </div>
              )}
            </div>
          </div>
        )}
      </div>
    );
  };

  const renderFieldInput = (field: FormField) => {
    const hasError = errors[field.id];

    return (
      <div
        key={field.id}
        className="bg-white rounded-lg border border-gray-200 p-4 sm:p-6 mb-4"
      >
        <label className="block text-base font-medium text-gray-900 mb-3 whitespace-pre-wrap">
          {field.label}
          {field.required && <span className="text-red-500">*</span>}
        </label>

        {field.type === "text" && (
          <div>
            <input
              type="text"
              placeholder="Text"
              value={formResponses[field.id] || ""}
              onChange={(e) => handleResponseChange(field.id, e.target.value)}
              className={`w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                hasError ? "border-red-500" : "border-gray-300"
              }`}
            />
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "longText" && (
          <div>
            <textarea
              placeholder="Long Text"
              rows={4}
              value={formResponses[field.id] || ""}
              onChange={(e) => handleResponseChange(field.id, e.target.value)}
              className={`w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                hasError ? "border-red-500" : "border-gray-300"
              }`}
            />
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "email" && (
          <div>
            <input
              type="email"
              placeholder="Email"
              value={formResponses[field.id] || ""}
              onChange={(e) => handleResponseChange(field.id, e.target.value)}
              className={`w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                hasError ? "border-red-500" : "border-gray-300"
              }`}
            />
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "number" && (
          <div>
            <input
              type="number"
              placeholder="Number"
              value={formResponses[field.id] || ""}
              onChange={(e) => handleResponseChange(field.id, e.target.value)}
              className={`w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                hasError ? "border-red-500" : "border-gray-300"
              }`}
            />
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "multipleChoice" && field.options && (
          <div>
            <div className="space-y-2">
              {field.options.map((option) => (
                <label
                  key={option.id}
                  className="flex items-center gap-2 cursor-pointer"
                >
                  <input
                    type="radio"
                    name={field.id}
                    value={option.value}
                    checked={formResponses[field.id] === option.value}
                    onChange={(e) =>
                      handleResponseChange(field.id, e.target.value)
                    }
                    className="w-4 h-4"
                  />
                  <span className="text-gray-700">{option.value}</span>
                </label>
              ))}
            </div>
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "checkboxes" && field.options && (
          <div>
            <div className="space-y-2">
              {field.options.map((option) => (
                <label
                  key={option.id}
                  className="flex items-center gap-2 cursor-pointer"
                >
                  <input
                    type="checkbox"
                    checked={(formResponses[field.id] || []).includes(
                      option.value,
                    )}
                    onChange={(e) =>
                      handleCheckboxChange(
                        field.id,
                        option.value,
                        e.target.checked,
                      )
                    }
                    className="w-4 h-4"
                    style={{ accentColor: "#7c3ead" }}
                  />
                  <span className="text-gray-700">{option.value}</span>
                </label>
              ))}
            </div>
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "dropdown" && field.options && (
          <div>
            <select
              value={formResponses[field.id] || ""}
              onChange={(e) => handleResponseChange(field.id, e.target.value)}
              className={`w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500 ${
                hasError ? "border-red-500" : "border-gray-300"
              }`}
            >
              <option value="">Select an option</option>
              {field.options.map((option) => (
                <option key={option.id} value={option.value}>
                  {option.value}
                </option>
              ))}
            </select>
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}

        {field.type === "fileUpload" && (
          <div>
            <div className="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center">
              <input
                type="file"
                onChange={(e) =>
                  handleResponseChange(field.id, e.target.files?.[0])
                }
                className="hidden"
                id={`file-${field.id}`}
              />
              <label
                htmlFor={`file-${field.id}`}
                className="px-4 py-2 text-purple-600 border border-purple-600 rounded-md hover:bg-purple-50 inline-flex items-center gap-2 mx-auto cursor-pointer"
              >
                <Upload size={16} className="text-purple-600" />
                <span>Add File</span>
              </label>
              {formResponses[field.id] && (
                <p className="text-sm text-gray-600 mt-2">
                  Selected: {formResponses[field.id].name}
                </p>
              )}
            </div>
            {hasError && (
              <p className="text-red-500 text-sm mt-1">{errors[field.id]}</p>
            )}
          </div>
        )}
      </div>
    );
  };

  // Loading View
  if (isLoading) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-center">
          <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-purple-600 mx-auto mb-4"></div>
          <p className="text-gray-600">Loading test...</p>
        </div>
      </div>
    );
  }

  // Fill Form View
  if (viewMode === "fillForm" && selectedForm) {
    return (
      <div className="min-h-screen bg-gray-50">
        <div className="bg-white border-b border-gray-200 px-4 sm:px-6 py-4">
          <div className="flex items-center gap-4">
            <button
              onClick={() => {
                setViewMode("builder");
                setSelectedForm(null);
                setFormResponses({});
                setErrors({});
              }}
              className="text-gray-600 hover:text-gray-900 flex items-center gap-2"
            >
              <ArrowLeft size={20} />
            </button>
            <h1 className="text-lg sm:text-xl font-semibold truncate">
              {selectedForm.title}
            </h1>
          </div>
        </div>

        {showSubmitSuccess && (
          <div className="fixed top-20 right-4 sm:right-6 bg-green-500 text-white px-4 sm:px-6 py-3 rounded-lg shadow-lg z-50">
            ✓ Form submitted successfully!
          </div>
        )}

        <div className="max-w-3xl mx-auto py-8 px-4 sm:px-6">
          {selectedForm.fields.map(renderFieldInput)}

          <button
            onClick={handleSubmit}
            className="mt-6 w-full sm:w-auto px-8 py-3 bg-purple-600 text-white rounded-md hover:bg-purple-700 font-medium"
          >
            Submit
          </button>
        </div>
      </div>
    );
  }

  // Preview View
  if (viewMode === "preview") {
    return (
      <div className="min-h-screen bg-gray-50">
        <div className="bg-white border-b border-gray-200 px-4 sm:px-6 py-4">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-3">
              {isPreviewMode ? (
                <button
                  onClick={() => {
                    const urlParams = new URLSearchParams(
                      window.location.search,
                    );
                    const from = urlParams.get("from");
                    if (from === "table") {
                      window.location.href = "/oncampus?tab=aptitude&view=list";
                    } else if (from === "dashboard") {
                      window.location.href =
                        "/oncampus?tab=aptitude&view=kanban";
                    } else {
                      window.history.back();
                    }
                  }}
                  className="text-gray-600 hover:text-gray-900 flex items-center gap-2"
                >
                  <ArrowLeft size={20} />
                </button>
              ) : (
                <button
                  onClick={() => setViewMode("builder")}
                  className="text-gray-600 hover:text-gray-900 flex items-center gap-2"
                >
                  <ArrowLeft size={20} />
                </button>
              )}
              <File size={24} className="text-purple-600" />
              <h1 className="text-lg sm:text-xl font-medium truncate">
                {formTitle}
              </h1>
            </div>

            <div className="flex items-center gap-4">
              {timeLimitMinutes && (
                <span className="text-sm text-gray-600">
                  Time Limit: {timeLimitMinutes} minutes
                </span>
              )}
              <span className="text-sm text-gray-600">
                Total Score: {totalScore} points
              </span>
              {/* Edit Button in Preview */}
              <button
                onClick={() => setViewMode("builder")}
                className="flex items-center gap-2 px-6 py-2 bg-purple-600 text-white rounded-md font-medium text-sm border border-transparent hover:bg-purple-700 transition-colors focus:outline-none focus:ring-2 focus:ring-purple-500"
                title="Edit"
              >
                <Edit size={18} />
                <span>Edit</span>
              </button>
            </div>
          </div>
        </div>

        <div className="max-w-3xl mx-auto py-8 px-4 sm:px-6">
          {fields.length === 0 ? (
            <div className="flex flex-col items-center justify-center py-16">
              <File size={48} className="text-gray-300 mb-4" />
              <div className="text-lg font-semibold text-gray-500 mb-2">No data</div>
              <div className="text-sm text-gray-400">No questions have been added to this test yet.</div>
            </div>
          ) : (
            fields.map(renderFieldPreview)
          )}
        </div>
      </div>
    );
  }

  // Builder View
  const hasTimeLimitValidationError = saveErrors.some((error) =>
    error.toLowerCase().includes("time limit"),
  );

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white border-b border-gray-200 px-4 sm:px-6 py-4">
        <div className="flex items-center justify-between">
          <div className="flex items-center gap-2 sm:gap-3">
            <button
              onClick={() => window.history.back()}
              className="p-2 hover:bg-gray-100 rounded-full"
              title="Go back"
            >
              <ArrowLeft size={20} />
            </button>
            <File size={24} className="text-purple-600" />
            <input
              type="text"
              value={formTitle}
              onChange={(e) => {
                setFormTitle(e.target.value);
                setHasUnsavedChanges(true);
                setIsFormSaved(false);
              }}
              className="text-lg sm:text-xl font-medium outline-none border-b border-transparent hover:border-gray-300 focus:border-purple-500 w-96"
              placeholder="Test Title"
            />
          </div>

          <div className="flex flex-wrap items-center justify-end gap-2 sm:gap-3">
            <div className="flex h-10 items-center px-1">
              <Clock size={16} className="text-gray-500" />
              <span className="ml-2 text-sm font-medium text-gray-600">Time Limit</span>
              <span className="mx-2 w-px" />
              <div
                className={`flex h-10 items-center rounded-md border px-2 ${
                  hasTimeLimitValidationError
                    ? "border-red-500 bg-red-50"
                    : "border-gray-300 bg-[#f8f8f9]"
                }`}
              >
                <input
                  type="number"
                  value={timeLimitMinutes || ""}
                  onChange={(e) => {
                    const value = parseInt(e.target.value);
                    if (e.target.value === "" || isNaN(value)) {
                      setTimeLimitMinutes(undefined);
                    } else {
                      setTimeLimitMinutes(Math.min(480, Math.max(1, value)));
                    }
                  }}
                  onKeyPress={(e) => {
                    if (!/[0-9]/.test(e.key) && e.key !== "Backspace") {
                      e.preventDefault();
                    }
                  }}
                  className="w-14 border-none bg-transparent text-center text-sm font-medium text-gray-800 outline-none"
                  placeholder="60"
                  min="1"
                  max="480"
                  title="Enter time limit in minutes (1-480)"
                />
                <span className="ml-1 text-sm text-gray-500">min</span>
              </div>
            </div>
            <span className="mx-2 h-5 w-px bg-gray-300" />
            <div className="flex h-10 items-center px-1">
              <Trophy size={16} className="text-gray-500" />
              <span className="ml-2 text-sm text-gray-600">Total Score</span>
              <span className="mx-2 w-px" />
              <div className="flex h-10 items-center rounded-md border border-gray-300 bg-[#f8f8f9] px-2">
                <span className="text-sm font-semibold text-gray-700">{totalScore}</span>
                <span className="ml-1 text-sm text-gray-500">Points</span>
              </div>
            </div>

            <button
              onClick={() => setViewMode("preview")}
              className="flex h-10 items-center gap-2 rounded-md border border-gray-300 bg-white px-4 text-sm font-medium text-gray-700 shadow-sm transition-colors hover:border-gray-400 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-[#7c3ead]"
              title="Preview"
            >
              <Eye size={16} className="text-gray-600" />
              <span>Preview</span>
            </button>

            <button
              onClick={saveForm}
              disabled={isSaving || isFormSaved}
              className={`flex h-10 items-center gap-2 rounded-md border px-4 text-sm font-medium shadow-sm ${
                isFormSaved
                  ? "cursor-not-allowed border-[#7c3ead] bg-[#7c3ead] text-white"
                  : "border-[#7c3ead] bg-[#7c3ead] text-white hover:bg-[#6a35a8] disabled:cursor-not-allowed disabled:opacity-60"
              }`}
            >
              {isSaving ? (
                <>
                  <div className="h-4 w-4 animate-spin rounded-full border-b-2 border-white"></div>
                  Saving...
                </>
              ) : isFormSaved ? (
                <>
                  <Save size={16} />
                  Saved
                </>
              ) : (
                <>
                  <Save size={16} />
                  Save
                </>
              )}
            </button>
          </div>
        </div>
      </div>

      {showSubmitSuccess && (
        <div className="fixed top-20 right-4 sm:right-6 bg-green-500 text-white px-4 sm:px-6 py-3 rounded-lg shadow-lg z-50">
          ✓ Test saved successfully!
        </div>
      )}

      {showLinkCopied && (
        <div className="fixed top-20 right-4 sm:right-6 bg-blue-500 text-white px-4 sm:px-6 py-3 rounded-lg shadow-lg z-50">
          ✓ Link copied to clipboard!
        </div>
      )}

      {saveErrors.length > 0 && (
        <div className="fixed top-20 left-4 sm:left-6 bg-red-500 text-white px-4 sm:px-6 py-3 rounded-lg shadow-lg z-50 max-w-md">
          <div className="font-semibold mb-2">Validation Errors:</div>
          <ul className="text-sm space-y-1">
            {saveErrors.map((error, index) => (
              <li key={index}>• {error}</li>
            ))}
          </ul>
        </div>
      )}

      <div className="flex h-[calc(100vh-73px)]">
        {/* Main Form Area - Full Width */}
        <div className="flex-1 p-4 sm:p-8 overflow-y-auto bg-gray-50">
          <div className="max-w-4xl mx-auto space-y-6">
            {/* Sections as Collapsible Cards */}
            
            {sections.map((section, index) => (
              <div
                key={section.id}
                className="relative flex gap-2"
              >
                <div className="flex items-start pt-5 text-gray-400 cursor-grab">
                  <GripVertical className="w-5 h-5" />
                </div>

                <div className="flex-1 rounded-lg border-2 border-dashed border-[#c4b5fd] bg-white overflow-visible">
                  <div className="flex items-center justify-between gap-3 px-4 py-4">
                    <div className="flex items-center gap-2 min-w-0 flex-1">
                      <div className="min-w-0">
                        <div className="text-xs text-gray-500 mb-0.5">
                          Section - {index + 1}
                        </div>
                        <div className="flex items-center gap-1">
                          <input
                            type="text"
                            value={section.title}
                            onChange={(e) =>
                              updateSectionTitle(section.id, e.target.value)
                            }
                            className="text-base font-medium text-gray-800 bg-transparent border-none outline-none focus:ring-0 p-0 min-w-0"
                          />
                          {/* <ChevronDown size={18} className="text-[#7c3aed] shrink-0" /> */}
                        </div>
                      </div>
                    </div>

                    <div className="flex items-center gap-2 shrink-0">
                      <span className="text-sm text-gray-600 border border-gray-300 bg-white px-3 py-1 rounded-full whitespace-nowrap">
                        Questions: {getSectionQuestionCount(section.id)}
                      </span>
                      <span className="text-sm border-[#7c3ead] bg-[#7c3ead] text-white px-3 py-1 rounded-full font-medium whitespace-nowrap">
                        Score: {getSectionScore(section.id)} Points
                      </span>
                      <button
                        type="button"
                        onClick={() => toggleSectionExpanded(section.id)}
                        className="p-1.5 text-gray-500 hover:text-gray-700 rounded-md hover:bg-gray-100"
                        title={isSectionExpanded(section.id) ? "Collapse" : "Expand"}
                      >
                        <ChevronsUpDown size={18} />
                      </button>
                      <div className="relative">
                        <button
                          type="button"
                          onClick={(e) => {
                            e.stopPropagation();
                            setOpenSectionMenuId(
                              openSectionMenuId === section.id
                                ? null
                                : section.id,
                            );
                          }}
                          className="p-1.5 text-gray-500 hover:text-gray-700 rounded-md hover:bg-gray-100"
                        >
                          <MoreVertical size={18} />
                        </button>
                        {openSectionMenuId === section.id && (
                          <div className="absolute right-0 top-full mt-1 z-20 w-48 bg-white border border-gray-200 rounded-lg shadow-lg py-1">
                            <button
                              type="button"
                              className="w-full text-left px-4 py-2 text-sm text-gray-700 hover:bg-gray-50"
                              onClick={() => setOpenSectionMenuId(null)}
                            >
                              Reorder Section
                            </button>
                            <button
                              type="button"
                              className="w-full text-left px-4 py-2 text-sm text-gray-700 hover:bg-gray-50"
                              onClick={() => duplicateSection(section.id)}
                            >
                              Duplicate Section
                            </button>
                            <button
                              type="button"
                              className="w-full text-left px-4 py-2 text-sm text-red-600 hover:bg-red-50"
                              onClick={() => deleteSection(section.id)}
                            >
                              Delete Section
                            </button>
                          </div>
                        )}
                      </div>
                    </div>
                  </div>

                  {isSectionExpanded(section.id) && (
                    <div className="px-4 pb-4 border-t border-gray-100">
                      <DndContext
                        sensors={sensors}
                        collisionDetection={closestCenter}
                        onDragEnd={handleDragEnd}
                        modifiers={[restrictToParentElement]}
                      >
                        <SortableContext
                          items={getSectionFields(section.id).map((f) => f.id)}
                          strategy={verticalListSortingStrategy}
                        >
                          {getSectionFields(section.id).length === 0 ? (
                            <div className="text-center py-8">
                              <FileDigit
                                size={48}
                                className="text-gray-300 mx-auto mb-3"
                              />
                              <p className="text-gray-500 font-medium">
                                No questions in this section yet
                              </p>
                              <p className="text-gray-400 text-sm mt-1">
                                Add your first question below
                              </p>
                            </div>
                          ) : (
                            getSectionFields(section.id).map((field, qIndex) => (
                              <SortableField
                                key={field.id}
                                field={field}
                                renderField={(f) =>
                                  renderFieldEditor(f, qIndex + 1)
                                }
                              />
                            ))
                          )}
                        </SortableContext>
                      </DndContext>

                      <button
                        type="button"
                        onClick={() => addField(section.id, "multipleChoice")}
                        className="w-full py-3 mt-4 border border-gray-300 rounded-lg text-[#7c3ead] hover:border-[#7c3aed] hover:bg-purple-50 font-medium flex items-center justify-center gap-2 transition-colors"
                      >
                        <Plus size={18} />
                        Add Question
                      </button>
                    </div>
                  )}
                </div>
              </div>
            ))}


            {/* Add Section Button */}
            <button
              onClick={() => setShowAddSectionModal(true)}
              className="w-full py-4 border-2 border-dashed border-purple-300 rounded-lg text-purple-600 hover:bg-purple-50 font-medium flex items-center justify-center gap-2 transition-colors"
            >
              <Plus size={20} />
              Add Section
            </button>
          </div>
        </div>

        {/* Add Section Modal */}
        {showAddSectionModal && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
            <div className="bg-white rounded-lg shadow-xl p-6 w-96 max-w-full mx-4">
              <h2 className="text-xl font-semibold mb-4">Add New Section</h2>
              
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Section Title
                  </label>
                  <input
                    type="text"
                    value={newSectionTitle}
                    onChange={(e) => setNewSectionTitle(e.target.value)}
                    placeholder="e.g., Data Structures"
                    className="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-500"
                  />
                </div>

                {/* <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Section Type
                  </label>
                  <select
                    value={newSectionType}
                    onChange={(e) => setNewSectionType(e.target.value as SectionType)}
                    className="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-500"
                  >
                    <option value="custom">Custom Section</option>
                    <option value="logic_reasoning">Logic Reasoning</option>
                    <option value="coding_assessment">Coding Assessment</option>
                    <option value="quantitative_ability">Quantitative Ability</option>
                  </select>
                </div> */}
              </div>

              <div className="flex gap-3 mt-6">
                <button
                  onClick={() => {
                    setShowAddSectionModal(false);
                    setNewSectionTitle("");
                    setNewSectionType("custom");
                  }}
                  className="flex-1 px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50 font-medium"
                >
                  Cancel
                </button>
                <button
                  onClick={addNewSection}
                  className="flex-1 px-4 py-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700 font-medium"
                >
                  Add Section
                </button>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
};

export default TestFormBuilder;
